# 11. Payment Gateway Integrations

## What this cluster covers

This cluster is about the code that actually talks to outside payment providers, the part of the system responsible for turning a domain order into a real charge against a real card or a real crypto wallet. Four folders live here, `stripe-integration`, `coingate-integration`, `cryptomus-integration`, and `ud-billing`, and the first three genuinely are payment gateways while the fourth, once you read it closely, turns out to be something else entirely.

Stripe is the traditional card processor, the one a frontend developer coming from a normal SaaS product will already recognize the shape of. Coingate and Cryptomus are both crypto payment processors, they let a buyer pay in Bitcoin, Ethereum, Litecoin and similar coins instead of a card, and reading them side by side with Stripe is the most useful thing this cluster offers, because the two families of provider genuinely need different data, confirm payment in different ways, and fail in different ways. `ud-billing` is not a fourth gateway at all, it is an internal, super admin only reporting and reconciliation module for the revenue share this company owes Unstoppable Domains on every UD domain it resells, and it reads records that the other three integrations, and the separate webhooks system, already produced.

## Files in this folder

[01-stripe-integration.md](01-stripe-integration.md) covers how a Stripe payment intent gets created, how a refund and a cancel actually work as raw HTTP calls, and the slightly unusual detail that this folder exposes no controller of its own at all.

[02-coingate-integration.md](02-coingate-integration.md) covers the hosted invoice flow, the crypto specific status list that includes states like `confirming`, and the multi step lookup Coingate refunds require before a refund request can even be sent.

[03-cryptomus-integration.md](03-cryptomus-integration.md) covers the other crypto processor, its own hosted invoice flow, the custom request signing scheme it uses on every outbound call, and the one place in this cluster where the code explicitly polls a provider for the current state of a payment rather than waiting on a webhook.

[04-ud-billing.md](04-ud-billing.md) covers what this module actually is, an admin analytics and revenue reconciliation dashboard for Unstoppable Domains orders, not a payment integration, and how it depends on data the other three components and the webhooks system already wrote.

[05-card-vs-crypto-comparison.md](05-card-vs-crypto-comparison.md) is the file worth reading even if you only care about payment systems in general and never touch this specific codebase again. It lines up real code from Stripe against real code from Coingate and Cryptomus to show, concretely, what a card integration needs that a crypto integration does not, and the reverse.

## The split with the webhooks cluster

Every one of the three real gateways in this folder creates a checkout session or invoice and hands the buyer a URL or a client secret to pay with, and every one of them also needs to find out, later, whether that payment actually succeeded. Creating the payment is this cluster's job. Finding out what happened to it is mostly not.

Each integration builds its outgoing request with a callback URL that points somewhere else in the codebase entirely. Coingate's invoice creation sets `callbackUrl` to `${BACKEND_API_URL}/coingate-webhook/coingateWebhook`, Cryptomus sets `url_callback` to `${BACKEND_API_URL}/cryptomusWebhook`, and Stripe's asynchronous status changes are read by a Stripe hosted webhook endpoint under the same pattern. All three of those addresses land inside `src/components/webhooks`, in the `stripe-webhook`, `coingate-webhook`, and `cryptomus-webhook` subfolders respectively, which this note did not read in detail since another set of notes covers that folder directly. What lives here, inside the integration folders themselves, is entity and repository methods with names like `updateByIntentId`, `updateCoingateStatusByToken`, and `updateStatusByOrderId`, plain database writers that some other piece of code is expected to call once it has verified a signature and parsed a payload. That other piece of code is the webhook controller in the neighboring folder, not anything living here. If you are trying to trace a payment status change all the way from the provider's signal to a row in Postgres, this cluster only gets you halfway, the signature check and the actual "this webhook just told us the invoice is paid" logic sits one folder over.

`ud-billing` is a partial exception to that split. Its repository reads directly from `WebhookAuditLogEntity`, a table that also lives under `src/components/webhooks` (`webhook-audit-log`), so this cluster does end up displaying webhook history to admins, it just does not write to that table or verify anything itself, it only ever reads.
