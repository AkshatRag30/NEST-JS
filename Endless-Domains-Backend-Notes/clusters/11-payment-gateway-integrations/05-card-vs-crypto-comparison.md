# 05. Card vs Crypto, Compared Directly

## Why this comparison is worth reading even outside this codebase

Most frontend developers moving into fullstack work will eventually touch a payment integration, and the overwhelming majority of the time it will be a card processor, Stripe most likely, sometimes Braintree or Adyen, all shaped roughly the same way. Far fewer will have touched a crypto payment processor before, and the honest truth is that the two are not the same problem wearing different branding. They need different data up front, they confirm success on fundamentally different timelines, and they fail in different ways. This codebase happens to implement three of these side by side, one card processor and two independent crypto processors, which makes it a genuinely good, concrete place to see the difference rather than take it on faith.

## What each one needs to even start

Stripe's payment intent creation needs almost nothing:

```ts
// stripe-integration.service.ts
const data = { amount: amount, currency: currency, 'automatic_payment_methods[enabled]': true };
```

Just an amount and a currency. Everything else about collecting payment happens later, client side, through Stripe's own JavaScript widget talking directly to Stripe using the `client_secret` this call returns. The buyer never leaves the page.

Coingate's order creation needs an amount too, but also this:

```ts
// coingate-integration.service.ts
createCoinGateOrderDto.callbackUrl = `${this.BACKEND_API_URL}/coingate-webhook/coingateWebhook`;
createCoinGateOrderDto.successUrl = `${this.WEB_CLIENT_URL}/payment/payment-success?&redirect_status=succeeded`;
createCoinGateOrderDto.cancelUrl = `${this.WEB_CLIENT_URL}/profile/orders`;
createCoinGateOrderDto.purchaserEmail = userData.email;
```

And Cryptomus needs the same shape of thing:

```ts
// cryptomus-integration.service.ts
url_success: `${this.WEB_CLIENT_URL}/payment/payment-success?redirect_status=succeeded`,
url_callback: `${this.BACKEND_API_URL}/cryptomusWebhook`
```

Both crypto processors need to know where to send the buyer's browser once they are done, because both of them are a hosted, off site checkout page, not an embeddable widget. That single architectural fact, hosted redirect versus embedded widget, is the reason a crypto integration's request payload is inherently a little larger and carries URLs a card integration never needs.

## How each one confirms a payment actually happened

This is the part that matters most, and the codebase's own status enums make the difference impossible to miss. Stripe's:

```ts
// stripe-status.enum.ts
export enum StripeStatusEnum {
    CREATED = 'CREATED', CANCELLED = 'CANCELLED', AMOUNT_CAPTURABLE_UPDATED = 'AMOUNT_CAPTURABLE_UPDATED',
    PARTIALLY_FUNDED = 'PARTIALLY_FUNDED', PAYMENT_FAILED = 'PAYMENT_FAILED',
    REQUIRES_ACTION = 'REQUIRES_ACTION', PROCESSING = 'PROCESSING', SUCCEEDED = 'SUCCEEDED'
}
```

`REQUIRES_ACTION` exists for things like 3D Secure, and `PROCESSING` exists too, so it would be wrong to say a card payment is always instantaneous down to the millisecond. But in the overwhelming majority of real card charges, the flow moves from `CREATED` to `SUCCEEDED` in well under a second, and nothing about this integration folder ever needs to actively ask "is it done yet", it waits for a webhook, once, and that webhook is expected almost immediately.

Cryptomus's status list tells an entirely different story:

```ts
// cryptomus.enum.ts
export enum PAYMENT_STATUS {
    PAID = 'paid', PAID_OVER = 'paid_over', WRONG_AMOUNT = 'wrong_amount', PROCESS = 'process',
    CONFIRM_CHECK = 'confirm_check', WRONG_AMOUNT_WAITING = 'wrong_amount_waiting', CHECK = 'check',
    FAIL = 'fail', CANCEL = 'cancel', SYSTEM_FAIL = 'system_fail', LOCKED = 'locked', ...
}
```

`process` and `check` are genuine, often minutes long waiting states, the buyer's transaction is sitting on a public blockchain and the system is watching it accumulate confirmations before it will call the payment real. `wrong_amount_waiting` describes a state that has no card equivalent at all, funds arrived, but not quite the expected amount yet, and the correct response is to wait and see rather than immediately succeed or fail. Coingate's own status list has the same shape, `new`, `pending`, `confirming`, and only then `paid`, with `confirming` naming the exact same underlying wait.

## The concrete, provable difference: polling shows up in the crypto code and nowhere in the card code

This is the single clearest, most literal difference this codebase actually shows, not just implied by the status enums but written directly into control flow. Inside `CryptomusIntegrationService.createCryptomusInvoice`, before deciding whether to reuse or replace an existing invoice, the code makes a live call back to the provider to ask what is actually true right now:

```ts
// cryptomus-integration.service.ts
const cryptomusDbOrder = await this.cryptomusInvoiceRepository.findByOrderIdNotFailed(orderId);
if (cryptomusDbOrder) {
    const paymentInfo = await this.getPaymentInfo({ uuid: cryptomusDbOrder.cryptomus_uuid });
    // branches on paymentInfo.status: SUCCESS, PENDING, or FAILED
}
```

`refundPayment` and `refundPartialAmount` both do the exact same live check again, right before attempting a refund, specifically because the local database's idea of the order's status might already be stale by the time a human clicks refund. Nowhere in `stripe-integration.service.ts` does an equivalent pattern exist. Stripe's `retrievePaymentIntent` can technically be called to fetch a fresh status, but nothing inside this codebase's own business logic calls it defensively before acting the way Cryptomus's `getPaymentInfo` gets called before every invoice reuse decision and every refund. That is not an accident of two different engineers writing similar code slightly differently, it is a direct consequence of card settlement being fast and reliable enough that a locally cached `payment_status` column can be trusted, while crypto settlement is slow and uncertain enough that the same local column has to be actively re verified against the source of truth before anything depends on it.

## Refunds: reversing a charge versus sending a new transaction

A Stripe refund needs almost nothing beyond identifying the original charge:

```ts
data.append('payment_intent', intentId);
data.append('amount', amount.toString());
```

Stripe already knows which card the money came from and reverses it there directly. A crypto refund cannot do that, there is no card network to ask, so both Coingate and Cryptomus require an explicit destination wallet address, supplied by the buyer, as part of the refund request itself:

```ts
// coingate-integration.service.ts
const data = { amount: coingateRefund.amount, address: coingateRefund.walletAddress, currency_id: ..., platform_id: ..., ledger_account_id: ... };
```

```ts
// cryptomus-integration.service.ts
const payload = { order_id: orderId, is_subtract: true, address: crytomusInvoice.from };
```

Coingate goes further still, requiring three separate lookup calls (currency id, platform id, and the merchant's own ledger account id) before it will even accept a refund request, because a crypto refund has to specify exactly which chain and which account to move funds from in a way a card refund simply never has to think about.

## Money units are not even consistent between providers

This company's own `totalCost` field is stored in cents throughout the domain order system. Stripe wants that same convention, cents, sent as a plain integer:

```ts
const data = { amount: amount, currency: currency, ... }; // amount is already in cents
```

Both crypto processors want dollars instead, and the integration code has to convert on the way out:

```ts
// coingate-integration.service.ts
createCoinGateOrderDto.priceAmount = orderDetails.totalCost / 100;
```

```ts
// cryptomus-integration.service.ts
amount: amount.toFixed(2), // amount was already divided by 100 by the caller
```

It is a small detail, but it is a real one, and it is exactly the kind of thing that causes a genuinely embarrassing bug (charging someone one hundred times too much, or one hundredth as much) if a future change to this code assumes every provider wants money expressed the same way just because they are all, nominally, "payment processors."

## The takeaway

A card integration's entire job is to hand off to a processor that already has decades of banking rails behind it, made to feel instant, and trusted to report back reliably and quickly. A crypto integration's job is to coordinate with a public, decentralized ledger that nobody, including the payment processor itself, has authority to speed up, which is why its status model has to include waiting states a card model never needed, why refunds need an explicit destination instead of an implicit reversal, and why, in this specific codebase, only the crypto integrations ever reach back out mid request to ask a provider "is this actually done yet" before making a decision that depends on the answer.
