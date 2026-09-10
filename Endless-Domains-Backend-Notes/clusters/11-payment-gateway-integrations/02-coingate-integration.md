# 02. Coingate Integration

## Coingate is a hosted checkout, not an embedded widget

The very first difference between this file and the Stripe note is visible in the shape of the request Coingate needs. `createCoingateOrder` builds this body:

```ts
// src/components/coingate-integration/coingate-integration.service.ts
const data = {
    order_id: createOrder.orderId,
    price_amount: createOrder.priceAmount,
    price_currency: createOrder.priceCurrency,
    receive_currency: createOrder.receiveCurrency,
    title: createOrder.title,
    token: createOrder.token,
    callback_url: createOrder.callbackUrl,
    cancel_url: createOrder.cancelUrl,
    success_url: createOrder.successUrl,
    purchaser_email: createOrder.purchaserEmail
};
const config = {
    method: 'post',
    url: this.COINGATE_API + CoinGateApiVersionEnum.version + CoinGateApiEnum.Order,
    headers: { Authorization: `Token ${this.COINGATE_KEY}`, 'Content-Type': 'application/x-www-form-urlencoded' },
    data: data
};
```

That is a `POST` to `{COINGATE_API}v2/orders`, authenticated with a `Token` scheme rather than Stripe's `Bearer`, and unlike Stripe's payment intent request it carries `success_url`, `cancel_url`, and `purchaser_email` up front. Coingate is going to hand back a `payment_url`, and the buyer's browser is going to be redirected to Coingate's own hosted page to actually pick a coin and pay, so Coingate needs to know in advance where to send that browser back to afterward, in either direction. Stripe never needed that because the buyer never leaves this company's own site.

`createCoingateInvoice` is the method that actually gets called from the controller, and it is worth reading for its idempotency handling as much as for the API call itself:

```ts
const getCoingateInvoiceByOrderId = await this.coingateIntegrationRepoInterface.findByOrderId(createCoinGateOrderDto.orderId);
if (getCoingateInvoiceByOrderId == undefined) {
    // create new invoice
} else {
    const coingateStatusListThatAllowNewInvoiceToBeCreate = [
        CoinGateStatusEnum.NEW.toString(), CoinGateStatusEnum.PENDING.toString(),
        CoinGateStatusEnum.CONFIRMING.toString(), CoinGateStatusEnum.EXPIRED.toString()
    ];
    if (coingateStatusListThatAllowNewInvoiceToBeCreate.includes(getCoingateInvoiceByOrderId.status)) {
        // delete old invoice, create and save a new one
    }
}
```

If a Coingate invoice already exists for this order, a brand new one only gets created when the existing one is still `new`, `pending`, `confirming`, or `expired`, any status that has not actually resolved into a real, final payment yet. If it is already `paid`, this code silently falls through and returns nothing, which is a real gap worth knowing about rather than a deliberate success path.

## A crypto specific status list

`CoinGateStatusEnum` is worth reading end to end, because it has no real equivalent on the Stripe side:

```ts
// src/components/coingate-integration/enum/coingate-status.enum.ts
export enum CoinGateStatusEnum {
    NEW = 'new', PENDING = 'pending', CONFIRMING = 'confirming', PAID = 'paid',
    INVALID = 'invalid', EXPIRED = 'expired', CANCELED = 'canceled',
    REFUNDED = 'refunded', PARTIALLY_REFUNDED = 'partially_refunded', ...
}
```

`confirming` is the one that matters most for the comparison this cluster is building toward. It represents a real, observable state a crypto payment passes through, the buyer has broadcast a transaction, Coingate has seen it show up, and now everyone is waiting for that transaction to accumulate enough blockchain confirmations before it can be trusted as final. Stripe's `StripeStatusEnum`, covered in the previous note, has nothing like it, a card charge simply succeeds or fails, it does not sit in an intermediate, still being confirmed state for minutes at a time the way a Bitcoin or Ethereum transaction does. This integration itself never has to poll to find out when `confirming` turns into `paid`, that transition is something Coingate's webhook, handled in `src/components/webhooks/coingate-webhook`, reports asynchronously, but the fact that the status enum needs a `confirming` value at all is a direct trace of the underlying blockchain settlement delay showing up in this codebase's data model.

## Refunds need Coingate to tell you who to pay and how

Stripe refunds needed only a payment intent id and an amount, because the money already knows which card to go back to. A crypto refund does not have that luxury, Coingate needs to be told which coin and which network to send the refund on, and that information has to be looked up first:

```ts
const getPayCurrency = await getPayCurrencyNameFromCoingateAPI(coingateInvoice.coingate_order_id, this.COINGATE_API, this.COINGATE_KEY);
const getCurrencyIdAndPlatformId = await getCurrencyIdAndPlatformIDFromCoingateAPI(getPayCurrency.currency_id, this.COINGATE_API);
const getLedgerAccountId = await getLeadgerAccountIdAPI(this.COINGATE_API, this.COINGATE_KEY);
```

Three separate `GET` calls, one to find out which currency and platform the buyer actually paid with, one to translate that currency symbol into the numeric ids Coingate's refund endpoint expects, and one to find the merchant's own ledger account id to refund from, all before the actual refund request can be built:

```ts
const data = {
    amount: coingateRefund.amount,
    address: coingateRefund.walletAddress,
    currency_id: getCurrencyIdAndPlatformId.currencyId,
    platform_id: getPayCurrency.platform_id,
    reason: coingateRefund.refundReason,
    email: userData.email,
    ledger_account_id: getLedgerAccountId,
    callback_url: `${this.BACKEND_API_URL}/coingate-webhook/coingateRefundWebhook`,
    order_id: coingateRefund.orderId,
};
```

`address` here is the buyer's own crypto wallet address, supplied directly by the frontend as part of `CreateCoinGateRefundDto`, because a crypto refund has to be sent somewhere explicit, there is no original card to automatically reverse the charge onto. Notice too that this refund request carries its own `callback_url`, the refund itself is asynchronous and Coingate reports back on it separately once it has actually gone out. `CoingateRefundEntity` stores that wallet address, the ledger account, and which chain the refund debited from (`balance_debit_currency_symbol`, commented as things like `ETH` or `LTC` or `XRP`), none of which Stripe's refund entity needed at all.

## What gets stored locally

`CoingateInvoiceEntity` (`tbl_coingate_invoice`) keeps `coingate_order_id`, `coingate_uuid`, `token`, `payment_url`, `price_amount`, and `status`. `updateCoingateInvoice` and the repository's `updateCoingateStatusByToken` exist for the same reason Stripe's `updateByIntentId` does, they are the write path a webhook handler elsewhere is expected to call once Coingate's own callback has arrived and been verified, not something this service calls on itself.
