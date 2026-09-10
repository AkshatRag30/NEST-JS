# 01. Stripe Integration

## The one detail to notice before anything else

Open the folder and look for a controller. There isn't one. Every other integration in this cluster, Coingate and Cryptomus both, ships its own `*.controller.ts` with real HTTP routes guarded by `AccessTokenGuard`. `stripe-integration` has none. `StripeIntegrationModule` exports `StripeIntegrationServiceInterface` and `StripeRefundServiceInterface` and nothing else, which means the only way anything in this codebase creates a Stripe payment is by another module, almost certainly the domain order module, injecting this service directly and calling it from inside its own controller. If you go looking for a `POST /stripe-integration/something` route expecting to find where a card payment starts, you will not find it here, you have to follow the injection outward instead.

## Creating a payment intent

```ts
// src/components/stripe-integration/stripe-integration.service.ts
private async callStripeAPI(amount: number, currency: string): Promise<any> {
    const apiKey = this.STRIPE_KEY;
    const data = { amount: amount, currency: currency, 'automatic_payment_methods[enabled]': true };

    const config = {
        method: 'post',
        url: this.STRIPE_URL + StripeAPIVersionEnum.V1 + StripeAPIEnum.PAYMENT_INTENT,
        headers: { Authorization: `Bearer ${apiKey}`, 'Content-Type': 'application/x-www-form-urlencoded' },
        data: data
    };
    const response = await httpClient.request(config);
    return response.data;
}
```

That builds a plain `POST` to `{STRIPE_URL}/v1/payment_intents`, authenticated with a `Bearer` token exactly the way you would authenticate against any other REST API, and the body is form encoded rather than JSON because that is what Stripe's own API expects. `automatic_payment_methods[enabled]: true` tells Stripe to decide for itself which payment methods to offer the buyer rather than the caller hardcoding `card` up front, which is why the response can come back naming Apple Pay or Google Pay later. Notice what is conspicuously absent from this payload compared to the crypto integrations covered in the next two notes, there is no success URL, no cancel URL, and no email address. Stripe's flow does not redirect the buyer anywhere, the frontend uses the returned `client_secret` directly with Stripe.js to render its own card element and confirm the payment in place.

`getStripeInfoFromAPI` wraps that call and maps the raw response onto `StripePaymentResponse`:

```ts
stripePaymentResponse.paymentClientId = response.client_secret;
stripePaymentResponse.paymentIntentId = response.id;
stripePaymentResponse.paymentMethod = response.payment_method_types[0];
stripePaymentResponse.paymentStatus = StripeStatusEnum.CREATED;
```

`paymentClientId` is what actually gets handed to the frontend, it is the `client_secret` a Stripe.js widget needs to finish collecting card details and confirm the charge without the backend ever seeing a raw card number. `createPaymentIntentWithMetadata` is a near identical second entry point that additionally appends `metadata[key]=value` pairs onto the same form body, letting the caller stash arbitrary key value pairs (an order id, most likely) onto the Stripe object itself so a later webhook can look them back up.

## Reading status back, cancelling, and figuring out the actual payment method

`retrievePaymentIntent` and the private `callStripeFetchIntentApi` do a plain `GET` on `/v1/payment_intents/{id}` and hand back `{ status, metadata }`. `cancelStripeIntentId` does a `POST` to `/v1/payment_intents/{id}/cancel`. Both are simple, synchronous, one shot calls, there is no waiting involved in either.

`fetchStripePaymentMethodUsingIntent` is more interesting because it exists at all. A `payment_intent` alone does not tell you whether the buyer actually paid with a card, Apple Pay, or Google Pay, Stripe surfaces that only on the underlying charge object, so this method fetches the intent first to get `latest_charge`, then fetches that charge with a second call to `/v1/charges/{id}`, and only then inspects `payment_method_details.card.wallet.type` to decide between `CARD`, `APPLE_PAY`, and `GOOGLE_PAY`:

```ts
const walletType = paymentMethodDetails?.card?.wallet?.type;
if (walletType) {
    switch (walletType) {
        case 'apple_pay': return DomainOrderPaymentMethodEnum.APPLE_PAY;
        case 'google_pay': return DomainOrderPaymentMethodEnum.GOOGLE_PAY;
        default: return DomainOrderPaymentMethodEnum.CARD;
    }
}
```

Every failure path in this method, a missing charge id, a charge id that does not look right, a failed fetch, quietly falls back to plain `CARD` rather than throwing. That is a deliberate choice to never let a display detail (which wallet a person tapped) block or break an otherwise successful payment record.

## Refunds are their own small service

`callStripeRefundAPI` posts to `/v1/refunds` with just `payment_intent` and `amount`:

```ts
const data = new URLSearchParams();
data.append('payment_intent', intentId);
data.append('amount', amount.toString());
```

There is no destination address to supply, unlike the crypto refunds covered later in this cluster, because Stripe already knows which card the original charge came from and simply reverses money back onto it. `getStripeRefundInfoFromAPI` wraps that call, and its `catch` block is worth reading closely because it deliberately does not always rethrow:

```ts
if (e.message == 'Failed to create refund') {
    stripeRefundResponseDto.status = StripeRefundStatusEnum.FAIL;
    return stripeRefundResponseDto;
}
throw new Error(`Stripe refund failed`);
```

A refund call that fails for the expected reason (Stripe rejected the refund) resolves normally with a `FAIL` status DTO instead of throwing, so the caller always gets a real object back to store, while any other, unexpected kind of error still propagates as an exception. `StripeRefundService`, in its own `stripe-refund` subfolder with its own entity, `tbl_stripe_refund`, and its own repository, is what actually persists that result, tagged with `refund_type` (`PARTIAL` or `FULL`) and `refund_status` (`SUCCESS` or `FAIL`).

## What gets stored locally

`StripeEntity` (`tbl_stripe`) stores `domain_order_id`, `price` (explicitly commented `amount is set in cent`, matching Stripe's own convention of working in the smallest currency unit), `payment_method`, `payment_client_id`, `payment_intent_id`, and `payment_status`. `StripeIntegrationRepoService.updateByIntentId` is the write path that a webhook handler elsewhere is expected to call once it has confirmed, out of band, that a given intent actually succeeded:

```ts
await this.stripeRepo.update(
    { payment_intent_id: intentId },
    { payment_status: status, payment_method: method }
);
```

Nothing inside this integration folder itself calls `updateByIntentId` in response to a successful payment. It exists to be called from outside, most likely from the Stripe webhook handler in `src/components/webhooks/stripe-webhook`, once that other code has verified Stripe's signature on the incoming event. That is the clearest example in this whole cluster of the split described in this cluster's README, this folder can create a payment and query its current state on demand, but the actual moment by moment status tracking is somebody else's job.
