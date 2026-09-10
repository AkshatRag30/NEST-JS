# 09. Events and Webhooks

## What this cluster actually covers

This cluster is about two different mechanisms that both use the word "event," and telling them apart clearly is the single most useful thing you can take from these notes.

The first mechanism is webhooks. A webhook is an HTTP endpoint this backend exposes so that an outside company's server, Stripe, Cryptomus, CoinGate, or Unstoppable Domains, can call it and say "something happened on our side." These endpoints live in `src/components/webhooks`, with a separate subfolder per provider, plus a shared `webhook-audit-log` table and a small `webhook-helper` of shared mutation functions used by all of them.

The second mechanism is this application's own internal event bus, built on top of Nest's `EventEmitterModule`. Code inside this backend calls `this.eventEmitter.emit('some.event.name', payload)` in one place, and a completely different piece of code elsewhere, decorated with `@OnEvent('some.event.name')`, reacts to it, usually to send an email or log something to an analytics tool. Nothing external is involved at all. The folders `src/components/event` and `src/components/event-subscribe` sit right next to a folder confusingly named `src/components/events`, which is not this event bus at all, it is an admin managed CMS for real world marketing conferences the company runs or attends. Note 03 untangles this naming collision in detail, because it trips people up on first read.

Rounding out the cluster, `src/components/subscribe` handles newsletter signups, and `src/components/mail-send-meta-data` is a thin, read only window onto a database table meant to record which emails have actually been sent to whom, though as note 05 explains, nothing in the current codebase actually writes to that table.

## Files in this cluster

`01-webhook-receivers-overview.md` walks through every webhook receiver in the codebase, what each one is for, and the shared shape they all follow (controller writes an audit log row, then hands the raw body to a service).

`02-signature-verification-deep-dive.md` is the most important file here. It compares, line by line, how each of the four payment webhook receivers does or does not prove that an incoming request genuinely came from the provider it claims to be from. The short version: one of them does this properly, one does it adequately, one computes a signature and then never checks it, and one does not attempt it at all. This file names each case plainly.

`03-internal-events-vs-external-webhooks.md` is the file that exists specifically to stop you from conflating "this app's internal event system" with "a webhook an external service sends us." It walks through a real, traceable example of each, end to end.

`04-payment-webhook-to-order-handoff.md` explains what happens after a webhook is trusted, how it finds the right order row in the database, and how it hands off to the commerce side of the app (domain orders, refunds, invoices) without this note pretending to be a tour of that other cluster.

`05-marketing-events-subscriptions-and-mail-audit.md` covers the CMS style `events` module, the `event-subscribe` and `subscribe` newsletter style modules, and the `mail-send-meta-data` audit table.

## Before you read further

If you have not yet read `02-high-level-architecture-and-bootstrap.md`, read it first. The line in `main.ts` that captures `req.rawBody` during body parsing is directly relevant to this cluster, and note 02 covers why it exists. `05-logging-and-observability.md` is also worth having fresh in memory, several of the webhook services call `customLoggerService.logWebhookEvent(...)`, and those calls are some of the most reliable markers of "this is a point in the code someone already decided mattered."
