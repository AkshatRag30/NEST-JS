# 01. Webhook Receivers Overview

## Why a domain marketplace needs webhooks at all

When a customer pays for a domain on this site, the payment itself often does not finish instantly inside the request that started it. A card payment through Stripe can take a moment to settle, a crypto payment through Cryptomus or CoinGate can take minutes or longer for the transaction to confirm on chain, and Unstoppable Domains' own claim and mint process runs as its own asynchronous operation on their servers. The browser cannot just sit there waiting. Instead, the provider calls back into this backend, on its own schedule, whenever the payment or operation actually changes state. That callback is a webhook, an HTTP POST this backend must expose publicly, with no logged in user attached to the request at all, because the caller is a server, not a person.

## The five receivers

`src/components/webhooks` contains five separate webhook receivers, each its own small Nest module:

`stripe-webhook` receives Stripe's card payment events, `payment_intent.succeeded`, `payment_intent.payment_failed`, and so on, for ordinary domain purchase orders.

`cryptomus-webhook` (a subfolder of `webhooks`) receives Cryptomus's crypto payment status callbacks, `paid`, `check`, `fail`, `cancel`, and the rest, for orders paid in crypto through Cryptomus.

`coingate-webhook` receives the equivalent status callbacks from CoinGate, a second crypto payment provider, on two separate routes, one for ordinary payment status and one specifically for refund status.

`udwebhook` is different in kind from the other three. It does not confirm a payment at all. It receives status callbacks from Unstoppable Domains' own v3 API about domain claim, mint, and transfer operations, the actual on chain work of handing a domain to its new owner once payment has already cleared through one of the other three.

`ai-subscription-webhook` (a subfolder of `webhooks`) is a smaller, newer addition, a second, independent Stripe integration used specifically for the AI advisor subscription product rather than domain purchases. It is worth reading side by side with `stripe-webhook`, because as note 02 covers, it verifies its signature properly while the older `stripe-webhook` does not verify anything at all.

## The shared controller shape

Every one of these controllers follows close to the same pattern. Here is the Cryptomus one in full, since it is short enough to show completely:

```ts
@Post('/')
@HttpCode(HttpStatus.OK)
public async cryptomusWebhook(@Body() reqBody: any): Promise<void> {
    const receivedAt = new Date();
    this.webhookAuditLogRepo.createAuditLog({
        provider: 'CRYPTOMUS',
        orderId: reqBody?.order_id ?? null,
        transactionId: reqBody?.uuid ?? null,
        eventType: reqBody?.status ?? null,
        status: 'RECEIVED',
        receivedAt,
        rawPayload: reqBody,
    }).catch(e => this.logger.error(`Cryptomus webhook audit log failed: ${e?.message}`));

    await this.cryptomusWebhookInterface.cryptomusWebhook(reqBody);
}
```

Two things about this shape are worth noticing on purpose, because they show up in every one of these controllers with only the field names changed. First, the very first thing the controller does, before any business logic runs at all, is write a row into `tbl_webhook_audit_log` recording that something arrived, what provider it claims to be from, and the entire raw payload. Second, that write is deliberately not awaited into the main flow in a way that could block it, it is fired with `.catch(...)` attached directly to it rather than wrapped in a `try` around the whole handler, so that if writing the audit log itself fails for some reason, that failure is only logged, it can never stop the actual webhook processing underneath it from running. Whoever built this clearly wanted a durable, best effort trail of every webhook that ever hit this server, succeeded or not, verified or not, even if the row sometimes only has a provider name and a timestamp because the body was malformed.

`WebhookAuditLogEntity`, the table this writes to, looks like this:

```ts
@Entity({ name: 'tbl_webhook_audit_log' })
export class WebhookAuditLogEntity extends BaseEntity {
    provider: string;      // STRIPE | CRYPTOMUS | COINGATE | UD
    orderId: string;
    transactionId: string; // Stripe paymentIntentId, Cryptomus UUID, Coingate UUID, UD operationId
    eventType: string;     // e.g. payment_intent.succeeded, paid, PAID_OVER, DOMAIN_CLAIM
    status: string;        // RECEIVED | SUCCESS | FAILED
    responseMessage: string;
    receivedAt: Date;
    rawPayload: any;       // jsonb
    correlationId: string;
    errorDetails: string;
}
```

Storing the entire raw payload as `jsonb` is what makes this table genuinely useful later, if a payment provider disputes what it sent, or a customer says their payment succeeded but the order never updated, someone can query this table directly and see the exact bytes that arrived, independent of whatever the rest of the code did with them.

After the audit log write, the controller hands the untouched request body straight to the corresponding service, `stripeWebhook(reqBody)`, `cryptomusWebhook(reqBody)`, `coingateWebhook(reqBody)`, `udV3Webhook(header, reqBody)`. That service is where the real work happens, matching the payload to an order, verifying it if the provider's design calls for verification, and moving the order forward.

## Two receivers wired outside the aggregating module

One structural detail worth knowing if you go looking for these later: `src/components/webhooks/webhooks.module.ts` only imports `UdWebhookModule`, `StripeWebhookModule`, and `CoingateWebhookModule`. `CryptomusWebhookModule` and `AiSubscriptionWebhookModule` are not imported there at all, they are registered directly in `app.module.ts` alongside `WebhooksModule` itself. Functionally this makes no difference, Nest wires all of them into the same application regardless, but it means the `webhooks` folder's own module file is not a complete map of what actually lives under it, you have to check `app.module.ts` too if you want the full list of five.
