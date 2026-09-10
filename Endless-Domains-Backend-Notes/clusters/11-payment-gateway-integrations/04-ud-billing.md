# 04. UD Billing

## What this actually is, once you read past the name

Sitting next to Stripe, Coingate, and Cryptomus, `ud-billing` looks like it should be a fourth payment gateway, something to do with Unstoppable Domains handling a charge. It is not. Every route on its controller is locked behind `SuperAdminAccessGuard`, and every method on its service is read only reporting or bookkeeping over data other parts of the system already wrote. There is no outbound HTTP call to any provider anywhere in this folder, no API key for talking to a payment processor, and no invoice or checkout of any kind. `ud-billing` is an internal finance and analytics dashboard, built specifically around the revenue share Endless Domains owes Unstoppable Domains on every UD domain it resells.

```ts
// src/components/ud-billing/ud-billing.service.ts
const UD_TLD_PERCENTAGE_MAP: Record<string, number> = {
    '888': 30, altimist: 20, anime: 20, austin: 30, bald: 20, ...
    bitcoin: 30, blockchain: 30, crypto: 30, nft: 30, unstoppable: 30, wallet: 30, ...
    og: 50,
};
const DEFAULT_UD_PERCENTAGE = 30;
```

That map is the whole business model of this module in one place, every top level domain (`.crypto`, `.nft`, `.wallet`, `.og`, and dozens more) UD lets this company resell carries a fixed cut UD is owed, thirty percent for most of them, fifty for `.og`, twenty or fifteen or ten for a long tail of smaller ones, and thirty as a fallback for anything not explicitly listed. Every report this module produces is ultimately built on top of `customerPaid * pct / 100`.

## The reports it actually generates

`generateBillingReport` pulls every completed UD domain order in a date range and, per row, computes what the customer paid, what UD's cut of that is, and what is left over as net margin, then rolls all of that up into totals and a summary. `generateTldAnalytics` groups the same underlying orders by TLD to answer a different question, which top level domains are actually selling, which have had zero sales since launch, and which are selling so rarely they are candidates for the company to stop offering. `generateGrowthInsights` goes further still, computing a month over month revenue trend, finding peak selling days and hours, finding repeat buyers, and flagging billing anomalies, duplicate domain names across separate orders, orders with a zero price, and completed orders that have no matching on chain transaction record at all, any of which would be worth a human looking at directly. `exportUdBilling` turns a date range into a real `.xlsx` file (via the `exceljs` package) formatted as `DOMAIN_CLAIM` and `DOMAIN_RETURN` rows with cost and amount owed columns, clearly built to be handed to UD itself or to this company's own finance team as a settlement document. `reconcileDomains` takes a plain list of domain names pasted in by an admin and matches it against what this system's own order history actually shows, splitting the result into domains it can confirm a sale for and domains it has no record of at all.

`generatePaymentAnalytics` is the one method in this file that reaches furthest into the rest of this cluster, because it groups revenue by `paymentMethod`, the exact string field the Stripe, Coingate, and Cryptomus integrations each set on a domain order (`DomainOrderPaymentMethodEnum.CARD`, `COINGATE`, `CRYPTOMUS`) when a payment is created. It then classifies every method as card or crypto by matching against a small keyword list and, if card payments make up more than half of all orders, appends a fully written out, ready made recommendation to add PayPal as an additional payment option, reasoning, expected impact, and integration options all included directly in the response object. That entire block is a business recommendation baked directly into an analytics endpoint's return value, not something a frontend is expected to compute itself.

## Where it borrows from the webhooks system

```ts
// src/components/ud-billing/ud-billing.repo.ts
import { WebhookAuditLogEntity } from '@components/webhooks/webhook-audit-log/entity/webhook-audit-log.entity';
```

`getWebhookLogs`, and the query builder behind it in `UdBillingRepo`, reads directly out of `WebhookAuditLogEntity`, filterable by `provider` (`STRIPE`, `CRYPTOMUS`, `COINGATE`, or `UD`), `orderId`, `transactionId`, and `status` (`RECEIVED`, `SUCCESS`, `FAILED`). That table is written elsewhere, inside `src/components/webhooks`, every time any of this company's webhook handlers receives and processes an event, and this module never writes to it, only reads from it, paginated, for an admin screen. It is the one place `ud-billing` genuinely overlaps with the rest of this cluster rather than standing entirely apart from it, an admin auditing whether a given Coingate or Cryptomus webhook actually arrived and succeeded is looking at data this folder displays but that the webhooks cluster alone is responsible for producing.

## How it actually relates to Stripe, Coingate, and Cryptomus

`ud-billing` sits downstream of all three real integrations rather than beside them. It does not create a payment, does not call Stripe, Coingate, or Cryptomus, and does not verify anything. What it does is read the `paymentMethod`, `totalCost`, `orderStatus`, and `domainProvider` columns those integrations (and the domain order module they write into) already populated on `tbl_domain_order`, along with the webhook history the webhooks cluster already recorded, and turn all of that into revenue share math, TLD performance analytics, and a downloadable settlement spreadsheet for a super admin. If Stripe, Coingate, and Cryptomus are the three ways a payment actually gets made in this system, `ud-billing` is the reporting layer that later asks what happened as a result, specifically through the lens of the one company, Unstoppable Domains, that this system owes a percentage of every relevant sale to.
