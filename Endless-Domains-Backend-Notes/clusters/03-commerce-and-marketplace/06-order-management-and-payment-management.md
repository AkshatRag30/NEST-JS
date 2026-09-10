# 06. Order Management and Payment Management, Two Admin Dashboards, and Where the Real Checkout Logic Actually Lives

## Read this file's boundary claim carefully before anything else

Neither `order-management` nor `payment-management` creates an order, charges a card, or processes a webhook. Both are entirely read facing, admin only reporting layers built on top of entities that other modules own and write to. If you came into this cluster expecting to find "the checkout service" here, this is the file that tells you where it actually is, and it is deliberately outside the boundary this note taking pass was assigned.

`DomainOrderEntity` (`src/components/domain/domain-order/entity/domain-order.entity.ts`) is the table both of these modules revolve around, and it is worth knowing its shape even though the service that writes it belongs to another cluster:

```ts
@Entity({ name: 'tbl_domain_order' })
export class DomainOrderEntity extends BaseEntity {
    orderNumber: string;
    domainProvider: string;
    orderStatus: string; // Pending, Completed, Cancelled, Failed, Rejected
    totalCost: number;   // stored as an integer, cents; every read divides by 100
    promoValue: number;
    promoApplied: boolean;
    promoCodeUsed: string;
    paymentMethod: string;
    paymentClientId: string;
    paymentIntentId: string;   // Stripe
    cryptomusUuid: string;     // Cryptomus
    transactionSecret: string;
    domainDetailList: DomainDetailEntity[]; // one row per domain in the order
    txnFee: number;
}
```

Every dollar figure this cluster's reporting queries ever produce comes from dividing `totalCost` by 100, seen literally in every SQL query in `OrderManagementRepo`, `SUM("totalCost") / 100.0 AS total_revenue_dollars`. The actual order creation, payment intent generation, and completion logic that populates this table lives in `src/components/domain/domain-order` and the top level `webhooks` and payment gateway integration folders, none of which this pass read in depth, on purpose, since another agent's slice covers them. What follows is what this cluster's own two modules do with that data once it exists.

## `order-management`, the admin sales dashboard

`OrderManagementService` and its repo are a thin read layer, every method is a query, nothing here mutates an order. `viewAllOrders` is the main paginated, filterable order list an admin dashboard's orders table would call, supporting search across order number, payment intent id, payment client id, and domain name all at once, plus filters on status, provider, payment method, user, and date range, all through one `QueryBuilder` chain with `leftJoinAndSelect('order.domainDetailList', 'domain')`.

The more interesting methods are the aggregate ones, because they show the kind of query that used to live on the frontend and got deliberately moved server side. `getOrderPeriodBreakdown`'s own comment says so directly:

```ts
/**
 * Replaces the raw view-all(limit:500) pull the dashboard charts used to fetch and
 * aggregate client side. Same [start,end] window the frontend previously computed
 * itself for the daily/weekly/monthly selector, aggregated in SQL instead of in the browser.
 */
```

That is a real, concrete example of a performance fix worth recognizing if you have ever built an admin dashboard yourself, pulling five hundred raw orders down to the browser just to group and sum them in JavaScript works fine at small scale and becomes a real problem the moment the table grows, and the fix is exactly what this method does, four parallel SQL aggregate queries (`totalsRows`, `revenueRows`, `statusRows`, `paymentRows`, using Postgres `date_trunc` and `FILTER (WHERE ...)` clauses) that return only the already summarized numbers the chart needs. `getOrderAnalytics` does the same thing for a KPI dashboard, today, this week, this month, and all time order counts and revenue, each with a percent change against the prior equivalent period computed in SQL rather than pulled raw and diffed client side.

`getExportsCsvData` is the one place this module does real, if small, data shaping work outside a pure query, it pulls orders in a date range (interpreting the incoming `startDate`/`endDate` as Unix seconds, `new Date(startDate * 1000)`), joins each one to its domain detail row by hand in a `Promise.all` loop rather than a SQL join, flattens the result, and hands it to the `json2csv` library, streamed back to the client as a `text/csv` attachment straight from the controller.

## `payment-management`, a payment gateway record browser

`PaymentUserService`/`PaymentUserRepo` do the equivalent job for the payment side rather than the order side, one endpoint each for browsing Cryptomus invoices, CoinGate invoices, Stripe invoices, and separately CoinGate, Cryptomus, and Stripe refund records, every one of them backed by an entity that belongs to its own dedicated payment gateway integration module, not this cluster (`CryptomusInvoiceEntity`, `CoingateInvoiceEntity`, `StripeEntity`, and their respective refund entities). This module's entire contribution is joining those gateway specific tables sideways against `DomainDetailEntity` and `User` so an admin screen can see, for one payment record, which domain and which user it actually belongs to, in one response.

The join itself is worth understanding, because it solves a genuinely awkward real world data typing problem, and the fix is explained directly in the code:

```ts
// order_id/user_id/userId are stored as text but need to join against real
// uuid columns. Casting them straight to ::uuid throws a Postgres runtime
// error (and 500s the whole query) the moment any row holds a non-UUID or
// empty string, so the cast is only applied to values that actually look
// like a UUID; anything else just fails to join instead of erroring.
private safeUuidCast(column: string): string {
    return `(CASE WHEN ${column} ~ '^[0-9a-fA-F]{8}-...' THEN ${column}::uuid END)`;
}
```

Because several of the payment gateway entities store `order_id` or `user_id` as a plain `text` column rather than a real `uuid` column (a sign these tables were likely added at different times, by different people, possibly under time pressure, exactly the kind of situational detail worth noticing in a large codebase), a naive join that casts the text straight to `uuid` would crash the entire query the moment a single row happened to have a malformed or empty value in that column. Wrapping the cast in a regex check first turns that from "the whole admin screen 500s" into "this one row just doesn't join, gracefully," a small defensive pattern worth remembering any time you are joining across tables whose typing discipline you cannot fully vouch for.

## Frontend note

Both of these modules are the backend half of an internal admin panel, not anything an ordinary customer's checkout page ever calls, gated behind `SuperAdminAccessGuard`. Their real lesson for a frontend developer moving toward fullstack work is about where aggregation and CSV generation belong, if you have ever built a dashboard by fetching a large raw list and reducing it in the browser, `getOrderPeriodBreakdown`'s own comment is describing the exact mistake you were making, and the fix, moving the grouping and summing into the database query itself, is usually both simpler to reason about and dramatically cheaper once real data volume shows up.
