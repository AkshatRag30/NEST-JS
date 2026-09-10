# 08. Refunds, and the Marketplace's Own, Separate Payment and Webhook Pair

## The refund module is mostly a read layer, with the write path living elsewhere

`RefundEntity` (`tbl_refund`) tracks one refund per order (`domain_order_id` is `unique`), with `total_amount`, `refund_amount`, `payment_method`, and a `refund_status` that defaults to `'PENDING'` and moves through `RefundStatusEnum` (`New`, `Pending`, `Processing`, `Rejected`, `Completed`). `RefundService.createRefundRecord` exists, but grepping the codebase shows nothing inside this cluster actually calls it, the real creation of a refund record happens from the webhook handlers in the top level `webhooks` cluster, the same pattern already seen with `PurchaseInvoiceServiceInterface` and `updateCouponUsageCount`, this module owns the shape of the data and the read side, another module decides when a refund actually gets written.

What this module does own outright is the query layer admins actually use to work refunds, `fetchAllRefundList` and its Cryptomus specific sibling, both hand built `QueryBuilder` joins against `User` and `DomainDetailEntity` to attach an email address and a domain name to a bare refund row, filterable by status, order, user, wallet address (`ILIKE` matched, so a partial wallet prefix works), payment method, and date range. It is worth noticing that `fetchAllRefundList`'s pagination is done in memory, `mappedData.slice(skip, skip + limitNumber)` after the full result set has already been pulled and mapped, while its Cryptomus specific sibling does the equivalent slicing at the database level with `.offset(skip).limit(limitNumber)` before execution, the same query pattern implemented two different ways a few lines apart in the same file, a small, genuine inconsistency worth knowing about if you ever need to add a fourth filter to either one and want to match the existing style.

## A second, entirely separate payment and webhook pair, easy to mistake for the first one

`src/components/marketplace/payment` and `src/components/marketplace/webhook` look, at a glance, like they might be this cluster's version of the top level `webhooks` folder other agents are covering, or a duplicate of the payment gateway integration clusters. They are not. Read closely, they exist for one specific, much smaller feature entirely of their own, paying to promote an already listed marketplace domain so it shows up more prominently, `isPromoted` on `DomainListing`.

```ts
// src/components/marketplace/payment/constants/promotion-plans.constant.ts
export enum PromotionPlan { HOUR_24 = '24h', DAY_7 = '7d' }
export const PROMOTION_PLAN_PRICES: Record<PromotionPlan, number> = { [PromotionPlan.HOUR_24]: 1, [PromotionPlan.DAY_7]: 5 };
export const PROMOTION_PLAN_HOURS: Record<PromotionPlan, number> = { [PromotionPlan.HOUR_24]: 24, [PromotionPlan.DAY_7]: 168 };
```

One dollar for twenty four hours of promotion, five dollars for a week, that is the entire product. `PaymentService.getOrCreateCryptomusInvoice` builds a Cryptomus invoice directly, with its own signature scheme (`createHash('md5').update(base64(payload) + CRYPTOMUS_TOKEN)`) and its own callback URL, `${BACKEND_API_URL}/marketplace/webhook`, distinct from wherever the primary checkout flow's Cryptomus callback lands. It even implements its own light idempotency, an open, non-failed invoice already on file for the same domain, user, and plan gets reused and its live status re-checked with Cryptomus rather than a brand new invoice being minted every time the same request repeats.

`WebhookService.handleWebhook`, on the receiving end, verifies the signature the same way, then drives `WebhookPaymentInvoice.status` through Cryptomus's own status vocabulary (`confirm_check`, `paid`, `paid_over`, `refund_paid`, `cancel`, `wrong_amount`), and this is where the cluster's most careful concurrency guarding shows up, worth reading as a companion piece to the reconciliation cron in `04`:

```ts
private async handlePromotionAndExpiry(data: any, invoice: WebhookPaymentInvoice): Promise<boolean> {
    if (!invoice.domain_id || !invoice.user_id) return false;
    const claimResult = await this.webhookPaymentInvoice.update(
        { order_id: data.order_id, status: Not(In(this.TERMINAL_INVOICE_STATUSES)) },
        { status: data.status, paid_at: paidAt }
    );
    if (claimResult.affected !== 1) return false; // already processed or missing, bail out silently
    // ...only now does the promotion actually get turned on...
}
```

The exact same claim-by-conditional-update trick from `03` and `04` reappears here, guarding against Cryptomus redelivering the same `PAID` webhook (a normal, expected behavior of most webhook providers, not a bug on their end) and this handler double extending the promotion's expiry or double sending the "your domain is now promoted" email. Once that claim succeeds, though, turning `isPromoted` on for the listing, updating the expiry date on both the payment log and the invoice, and sending the confirmation email are, once again, four separate un-transacted statements in a row, the same repeated pattern this cluster's notes keep flagging. A crash between the claim succeeding and `isPromoted` actually flipping true would leave a customer who was just charged for a promotion with no promotion active, and because the invoice's status is now terminal (`'paid'`), nothing revisits that row afterward to notice or fix it, it would need a human to catch.

Overpayment and underpayment both get their own explicit, real world handling here too, `PAID_OVER` computes exactly how much the customer overpaid and, if that overage is more than fifty cents, flags the invoice `refund_email_sent` and emails the customer asking them to confirm their refund details, while `WRONG_AMOUNT` (underpayment) does the mirror image, computing the shortfall and asking for the same. Both guard their own "already sent" flag with the identical conditional update pattern, so a redelivered webhook cannot trigger a second refund request email.

## Frontend note

If you are building the UI for "promote this listing," the price list is genuinely just two numbers and two durations, but everything behind that simple choice, the invoice reuse logic, the webhook signature check, the claim guards against duplicate delivery, the overpayment and underpayment email flows, is real production grade payment handling in miniature, worth studying specifically because it is small enough to read in full in one sitting while still containing nearly every hard problem a bigger payment integration has to solve. And the boundary lesson matters just as much as the mechanics, this codebase has at least two independent Cryptomus integrations living in two different folders for two different products, which is exactly the kind of thing worth checking for explicitly (which webhook URL does this actually call, which credentials does it actually sign with) before assuming any two payment related files that look similar are actually the same code path.
