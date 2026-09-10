# 03. Cryptomus Integration

## A second hosted crypto checkout, with its own request signing

Cryptomus follows the same overall shape as Coingate, a hosted page the buyer gets redirected to, but it authenticates its outbound API calls in a genuinely different way. Instead of a static bearer token or a static `Token` header, every single request carries a signature computed fresh from that request's own body:

```ts
// src/components/cryptomus-integration/cryptomus-integration.service.ts
getSignature(data: Record<string, string | number | boolean>): string {
    return createHash('md5')
        .update(Buffer.from(JSON.stringify(data)).toString('base64') + this.CRYPTOMUS_TOKEN)
        .digest('hex');
}

private createCryptomusHeaders(data: Record<string, string | number | boolean>): Record<string, string> {
    return {
        'Content-Type': 'application/json',
        merchant: this.CRYPTOMUS_MERCHANT_ID,
        sign: this.getSignature(data)
    };
}
```

The body gets JSON stringified, base64 encoded, and the merchant's secret token is appended to that string before it is MD5 hashed, and the resulting hex digest travels in a `sign` header alongside a plain `merchant` id header. This is the same basic idea behind how a webhook payload gets signature checked on the way in, except here it is happening on the way out, every request this service sends has to prove it actually came from someone holding the shared secret, not just present a static key the way Stripe's bearer token or Coingate's token header do. It means every one of Cryptomus's outbound calls, order creation, payment lookup, and refund, calls `createCryptomusHeaders(payload)` with that specific call's own body, since a signature computed for one payload is worthless for another.

## Creating the invoice

```ts
const payload: CreatePaymentParams = {
    order_id: orderId,
    accuracy_payment_percent: 5,
    subtract: 100,
    amount: amount.toFixed(2),
    currency: 'USD',
    additional_data: '',
    is_payment_multiple: false,
    lifetime: 1800,
    url_success: `${this.WEB_CLIENT_URL}/payment/payment-success?redirect_status=succeeded`,
    url_callback: `${this.BACKEND_API_URL}/cryptomusWebhook`
};
const response = await httpClient.post<CryptomusPaymentResponse>(`${this.CRYPTOMUS_API}/payment`, payload, {
    headers: this.createCryptomusHeaders(payload)
});
```

`lifetime: 1800` gives the invoice a thirty minute window to be paid before Cryptomus expires it, a concept that simply does not exist for a Stripe payment intent, which stays open indefinitely until it is confirmed or explicitly cancelled. `accuracy_payment_percent: 5` and `subtract: 100` are both about the reality of paying in a volatile asset, `accuracy_payment_percent` tells Cryptomus how much underpayment or overpayment (from exchange rate drift between invoice creation and payment) still counts as a match, and `subtract` controls whether the network's own transaction fee gets taken out of the amount the merchant receives rather than added on top for the buyer. Neither of those concerns has a card payment equivalent, a card charge is exact to the cent and the card network's fees are a separate, invisible settlement detail the merchant never has to configure per transaction.

## The one place in this cluster that explicitly polls the provider

Before creating a brand new invoice, `createCryptomusInvoice` checks whether one already exists for this order, and if it does, it does not just trust whatever status was last written to the local database, it goes back to Cryptomus and asks:

```ts
const cryptomusDbOrder = await this.cryptomusInvoiceRepository.findByOrderIdNotFailed(orderId);
if (cryptomusDbOrder) {
    const paymentInfo = await this.getPaymentInfo({ uuid: cryptomusDbOrder.cryptomus_uuid });
    if (paymentInfo) {
        if (Object.values(PAYMENT_STATUS_SUCCESS).includes(paymentInfo.status)) {
            // ...persist the now known final status
            throw new BadRequestException('Payment already done');
        }
        if (Object.values(PAYMENT_STATUS_PENDING).includes(paymentInfo.status)) {
            return { invoice: paymentInfo.url };
        }
        if (Object.values(PAYMENT_STATUS_FAILED).includes(paymentInfo.status)) {
            // ...mark as failed
            throw new BadRequestException('Payment failed');
        }
    }
}
```

`getPaymentInfo` is itself a `POST` to `{CRYPTOMUS_API}/payment/info`, a synchronous, on demand status check against the provider, made from inside a request that a buyer's own browser triggered by revisiting the checkout. That is a genuine, direct instance of the polling behavior this cluster's comparison note focuses on, code deliberately asking the provider "what is the real state of this payment right now" rather than trusting a locally cached status or waiting passively for a webhook to arrive. Nothing in the Stripe integration folder ever does this as part of its own business logic, Stripe's near instant confirmation means the local `payment_status` column is trusted to already reflect reality once a webhook has landed, there is no reason to double check it live on every subsequent request the way Cryptomus does here.

The status list itself is a further, direct trace of blockchain settlement reality. `PAYMENT_STATUS` includes `wrong_amount`, `wrong_amount_waiting`, `process`, `confirm_check`, `check`, and `locked`, on top of the plain `paid`, `fail`, and `cancel` a card payment might need. `wrong_amount_waiting` in particular describes a state a card payment cannot meaningfully have at all, the buyer sent a transaction, it landed on chain, but the amount that actually arrived does not match what was expected closely enough yet, and the system has to wait to see whether more funds show up before deciding whether the payment failed outright.

## Refunds require the same live status check before they are attempted

```ts
async refundPayment(orderId: string): Promise<string> {
    const crytomusInvoice = await this.getPaymentInfo({ orderId, order_id: orderId });
    if (crytomusInvoice.status != PAYMENT_STATUS['PAID']) {
        if (Object.values(PAYMENT_STATUS_PENDING).includes(crytomusInvoice.status)) throw new BadRequestException(`Order is still processing`);
        throw new BadRequestException(`Order is ${crytomusInvoice.status}`);
    }
    const payload = { order_id: orderId, is_subtract: true, address: crytomusInvoice.from };
    const response = await httpClient.post(`${this.CRYPTOMUS_API}/payment/refund`, payload, { headers: this.createCryptomusHeaders(payload) });
}
```

Just like Coingate, the refund destination, `address`, comes from the payment record itself (`crytomusInvoice.from`, the wallet the original payment arrived from) rather than from any card network automatically reversing a charge. And just like the invoice creation path above, this refund call refuses to proceed on stale local data, it calls `getPaymentInfo` fresh, right before acting, to make sure the order is genuinely `PAID` at this exact moment and not merely `process` or `check`, still waiting on confirmations. `refundPartialAmount` is the same flow with an explicit dollar `amount` field added to the payload for a partial rather than full refund.

## What gets stored locally

`CryptomusInvoiceEntity` (`tbl_cryptomus_invoice`) stores `cryptomus_uuid`, `amount`, `invoice_currency`, `payer_currency`, `network`, `from` (the buyer's paying wallet address), `txid`, `is_final`, and `status`. `payer_currency`, `network`, and `txid` have no equivalent anywhere on the Stripe side, a card payment has no network to name and no transaction hash, a crypto one genuinely needs to record which chain it settled on and which on chain transaction proved it.
