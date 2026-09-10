# 04. From a Trusted Webhook to an Updated Order

This note is about the handoff point, what a webhook service does once it has a payload it is willing to act on, and where it hands control to the commerce side of the app. It is not a tour of the domain order or payment logic itself, that lives in its own cluster, this is deliberately just the seam between the two.

## Step one, find the order the webhook is talking about

Every provider identifies "which order is this about" differently, because none of these providers know or care about this app's own primary keys, they only know about the identifiers they themselves handed out earlier when the payment was created.

Stripe webhooks arrive with a payment intent id, `data.data.object.id`, and `stripe-webhook.service.ts` uses that to look up a locally stored mapping first, before it ever touches the order table:

```ts
const stripeDataAccordingToClientIntentId = await this.stripeIntegrationServiceInterface.readByIntentId(data?.data?.object?.id);
if (stripeDataAccordingToClientIntentId != undefined) {
    const domainOrderWithDetails = await this.domainOrderRepoInterface.findOneByIdWithDetails(stripeDataAccordingToClientIntentId.domain_order_id);
```

Cryptomus webhooks are simpler here, because Cryptomus is told this app's own order id up front when the payment session is created, so it just echoes it straight back in `body.order_id`:

```ts
const order = await this.domainOrderRepoInterface.findOneByIdWithDetails(body.order_id);
```

CoinGate webhooks carry a `token` instead, which resolves to a previously stored CoinGate invoice row, and that row is what carries the actual `order_id`:

```ts
const coingateInvoiceData = await this.coingateIntegrationServiceInterface.readByToken(body.token);
...
const orderStatus = await this.domainOrderServiceInterface.getDomainOrderWithDetails(coingateInvoiceData.order_id);
```

UD webhooks are the odd one out, as note 01 mentions, because they are not confirming a payment, they are confirming a domain claim, mint, or transfer operation that only happens after a payment from one of the other three has already succeeded, so their lookup path runs against domain order and domain detail records already marked as processing, not against a fresh, unpaid order.

In every case, the pattern is the same shape: take whatever identifier the provider hands back, resolve it through a small lookup table this app maintains from when the payment was first created, and arrive at a `DomainOrderEntity`, the same entity the commerce cluster owns and works with everywhere else in the app.

## Step two, move the order through its status machine

`DomainOrderStatus` is an enum this cluster's webhook services read and write constantly without owning, `PENDING`, `PROCESSING`, `COMPLETED`, `CANCELLED`. A webhook service's entire job, once it has the right order, is deciding which of those states the order should move to next, based on what the provider just reported, and then calling `this.domainOrderRepoInterface.save(order)` to persist it. For example, from `stripe-webhook.service.ts`, on a successful ENS domain transfer:

```ts
latestDomainOrderWithDetails.orderStatus = DomainOrderStatus.COMPLETED;
...
await this.domainOrderRepoInterface.save(latestDomainOrderWithDetails);
```

or, on a smart contract failure it needs to unwind:

```ts
latestDomainOrderWithDetails.orderStatus = DomainOrderStatus.CANCELLED;
...
await this.domainOrderRepoInterface.save(latestDomainOrderWithDetails);
```

This is the actual handoff point to the commerce cluster in a nutshell, the webhook service is a client of `DomainOrderRepoInterface` and `DomainOrderServiceInterface`, exactly the same interfaces the commerce cluster's own controllers use, there is no separate, webhook only copy of an order's status logic. Whatever invariants and side effects the commerce cluster expects around an order's status changing (this note deliberately does not describe them, that is the commerce cluster's own notes to cover) apply here too, because it is the same save call.

## Step three, everything that follows a status change

Once an order's status actually changes, three more things typically happen in the same webhook service method, worth knowing about so you recognize them on sight elsewhere.

Refunds get triggered and recorded through a shared, cross provider table. `RefundEntity`, created through `RefundServiceInterface.createRefundRecord(...)`, is deliberately provider agnostic, a refund started because of a Stripe failure and a refund started because of a Cryptomus cancellation both land in the same table, tagged with a `payment_method` field, so that anyone querying refunds does not need to know or care which of the four payment providers was involved.

Invoices get generated through three separate invoice services, `SalesInvoiceServiceInterface` when an order completes, `PurchaseInvoiceServiceInterface` right after that, and `RefundInvoiceServiceInterface` when a refund is issued. These calls are wrapped in their own `try/catch` blocks specifically so that an invoice PDF generation failure never prevents the order itself from being marked complete or the customer from being notified, you can see this reasoning spelled out directly in a comment in `stripe-webhook.service.ts`:

```ts
} catch (error) {
    this.customLoggerService.error(`Sales invoice generation failed for order ${latestDomainOrderWithDetails.id}, continuing with order completion notification`);
    this.customLoggerService.error(error);
}
```

And internal events get raised, `DOMAIN_ORDER_COMPLETED`, `STRIPE_REFUND_INITIATED`, `LOGSNAG_TRACK`, exactly the mechanism covered in note 03, so that the customer facing email and the internal analytics tracking both happen without the webhook service itself needing to know how an email gets sent or how LogSnag's API works.

## Where the audit log fits into this timeline

It is worth being precise about timing here, since note 01 covers the audit log write but this note covers the order lookup and status change that come after it. The audit log row is written the moment the request arrives, before signature verification (where it exists at all), before the order lookup, before any of this. That is intentional, it means the audit trail captures every webhook that ever hit the server, including malformed ones, ones that failed signature checks, and ones for orders that no longer exist, none of which is true of the order and status data described in this note, which only ever gets touched once a payload has passed whatever verification that particular receiver performs and has been matched to a real order row.
