# 05. Cancelled Order Diagnostics, an Automated Root Cause Analysis Tool

## What problem this actually solves

Somewhere in this system, orders on the primary registration flow (`DomainOrderEntity`, the cart and checkout business, not the on chain marketplace covered in `03` and `04`) get cancelled, most often by a cron elsewhere that gives up on an order that has sat `Pending` for too long. When that happens, someone on the support or engineering side needs to answer a genuinely hard question fast, did the customer actually pay, and if they did, where did their money and their domain actually end up. Answering that by hand means manually cross referencing the order row against Stripe's own records, CoinGate or Cryptomus webhook logs, a separate webhook audit log table, Unstoppable Domains operation rows, Freename lifecycle rows, ENS mint records, and CloudWatch logs, by hand, for every single cancelled order that gets escalated. `cancelled-order-diagnostics` is a purely read only module that does exactly that cross referencing automatically and hands back a structured verdict.

This is entirely diagnostic tooling, nothing in this module writes to the database, `CancelledOrderDiagnosticsRepo`'s constructor injects eight different repositories and reads from every one of them.

## Three endpoints, one investigation

`GET /order/diagnostics/lookup` takes a fuzzy `orderId`, `orderNumber`, or `domainName` and returns matching orders, a search box for support staff who only have a customer's complaint to go on. `GET /order/diagnostics/:orderId/diagnosis` is the real tool, it pulls the order plus every related record in parallel (`Promise.all` across the Stripe record, CoinGate webhooks, Cryptomus webhooks, the webhook audit log, Unstoppable Domains rows, Freename lifecycle rows, and any Stripe refund), builds a chronological timeline, and produces a best guess root cause with a confidence label. `GET /order/diagnostics/:orderId/logs` takes that same order's creation and cancellation timestamps and queries the two CloudWatch log groups described in the logging cluster's notes (one for errors, one for info) for anything mentioning this order's id in that exact window, giving a direct line from a diagnosed order straight to the actual application log lines from when it happened.

## Reading the signal, not just the status

The most instructive part of this module is `buildRcaSignals`, which does not trust any single system's status field on its own, it deliberately cross checks them against each other. The comment guarding the payment method branch explains a real, easy to get wrong trap:

```ts
// "card" is how Stripe card payments are stored in paymentMethod.
// Do NOT use paymentIntentId here - a Cryptomus/CoinGate order can have a paymentIntentId
// from an earlier abandoned Stripe attempt; using it would misroute the payment block.
if (pm.includes('stripe') || pm === 'card') { ... }
```

A user can start a checkout with Stripe, abandon it, come back and pay with Cryptomus instead, and the order row can still be left holding a `paymentIntentId` from that first abandoned attempt. Routing the diagnosis purely by "does a `paymentIntentId` exist" instead of by the order's actual, final `paymentMethod` would misread that entirely ordinary retry as a Stripe failure. This kind of guard, reasoning about how the data can legitimately end up in a confusing shape rather than just reading the obvious field, runs throughout this file.

The clearest example of genuinely clever cross referencing is how it tells a webhook handler crash apart from a webhook correctly reporting bad news:

```ts
const KNOWN_EXPECTED_EVENT_TYPES = new Set([
    'COINGATE_INVOICE_EXPIRED', 'COINGATE_CANCELED', 'CANCELLED', 'PAYMENT_FAILED', /* ...many more... */
]);
const actualProcessingFailures = failedAudit.filter(l => l.eventType && !KNOWN_EXPECTED_EVENT_TYPES.has(l.eventType));
const webhookProcessingFailed = actualProcessingFailures.length > 0;
```

An audit log entry marked `FAILED` does not necessarily mean the backend's own code threw an exception, most of the time it means the webhook correctly told this system the customer's payment failed or their checkout expired, which is a normal, expected business outcome, not a bug. Only an event type outside that known, expected set counts as a real processing failure worth calling out as an actual system problem. Getting this distinction wrong in either direction would either bury real bugs in a sea of expected cancellations, or wrongly alarm engineers over customers who simply changed their mind at checkout.

A second good example, ENS failures get split into two genuinely different causes using one field that is easy to overlook:

```ts
// transactionId IS NOT NULL = tx was submitted -> RegisterFailed event emitted on-chain
// transactionId IS NULL     = tx never submitted -> JavaScript exception during ENS flow
```

Whether a mint record's `transactionId` is present or null tells you whether the failure happened on chain (the transaction was submitted and the smart contract itself rejected it, worth checking gas or contract state) or in application code before anything ever reached the blockchain (an exception, worth checking a stack trace instead). Same surface symptom, "ENS registration failed," two completely different places to actually go look for the fix, and the tool tells you which one applies without anyone having to dig.

## `determineRootCause`, an ordered decision tree

Once the signals exist, `determineRootCause` walks them in a specific, deliberate priority order, each with a `confidence` label (`verified`, `high`, `medium`, `low`) that is itself useful information, not decoration. Awaiting 3DS bank authentication is checked first and short circuits everything else, since it can apply even to an order that has not been cancelled yet. Completed and still pending orders get an early, simple answer. Then, in order: payment captured but domain registration failed downstream (with UD, Freename, and ENS each getting their own more specific sub message using the enhancement checks described above), customer genuinely abandoned payment (verified against the payment provider's own terminal status, not guessed), the payment provider itself cancelled or expired the transaction, the webhook handler itself threw an unhandled exception, a domain validation failure with no confirmed payment behind it, a payment that plainly failed at the provider, a payment captured for a blockchain the fulfillment backend does not implement yet (flagged `verified` and explicitly worded "manual refund required," a genuinely serious real world state this tool exists specifically to surface fast), no payment notification ever arrived at all, and only after every one of those is ruled out, a generic timeout fallback.

That six and a half numbered case, `paymentCapturedProviderNotImplemented`, is worth sitting with for a moment, because it is the sharpest possible illustration of why this whole cluster's notes keep returning to the idea of a half completed payment. A customer's card was charged successfully, in full, and the backend has no code path at all to give them what they paid for on that particular chain. Nothing about that state resolves itself, no cron retries it, no webhook eventually fixes it, it sits there until a human looks. This diagnostics tool exists largely to make sure that particular case, and the several other genuine failure categories above it, get found and named quickly rather than discovered by accident weeks later.

## Frontend note

Nothing here is customer facing, this is an internal support and engineering tool, gated behind `SuperAdminAccessGuard` on every route. The reason it belongs in a frontend developer's notes at all is that it is the clearest possible illustration of the other side of a payment failure your checkout UI shows as one flat error message, "your order was cancelled." Behind that one sentence sits a dozen genuinely distinct underlying situations, a card the customer chose not to finish paying with, a webhook that never arrived, a payment that succeeded but a downstream registration that failed, a payment captured for a feature the backend never finished building, and this module is the tool built specifically to tell those apart after the fact, because the checkout UI itself, correctly, never tries to.
