# 08. Internal Search Analytics and Generic Event Tracking

## Two folders that share nothing except sitting near each other

`search-management` and the plain `analytics` folder end up in one file here because they are both, in their own way, easy to misread from their names alone, and because neither one connects to any of the Google products covered in the previous two files. `search-management` sits one folder away from `search-console` in the file tree and shares nothing with it whatsoever, it has no dependency on Google, no service account, no external API call of any kind. `analytics` sits one folder away from `google-analytics`, and again shares nothing, no Google client, no OAuth, no service account. Reading both in full rather than assuming either one extends its neighbor is exactly the kind of check this whole cluster's notes were built to model.

## `search-management`: what people actually type into this site's own search bar

Every route on `SearchManagementController` reads from `DomainSearchLogEntity`, an entity that belongs to the `domain` module rather than to this folder, and that entity is populated whenever a real visitor types a domain name into Endless Domains' own search box to check availability. This is first party product analytics about this company's own core feature, not a wrapper around any external search product.

Reading the whole controller in order tells a story about a feature that grew in two distinct phases. The older half, `viewTodaySearchDomainList`, `viewWeeklySearchDomainList`, `viewMonthlySearchDomainList`, `viewDailyTop10TldSearched`, `viewWeeklyTop10TldSearched`, and `viewMonthlyTop10TldSearched`, all pull a flat, unfiltered date range straight out of the database and then do all of their grouping, counting, deduplicating, and sorting in plain TypeScript after the fact. Every one of these six routes is gated by `SuperAdminAccessGuard`.

The newer half, `getSearchHistory`, `getTopTlds`, `getSearchTrend`, `getTldChart`, and `getSearchSummary`, pushes that same kind of work down into the database instead, using `QueryBuilder` with real `WHERE`, `GROUP BY`, and pagination clauses, and one of them, `getSearchSummary`, uses a single raw SQL query with Postgres's `FILTER (WHERE ...)` syntax to compute five different conditional counts (total, valid, invalid, available, unavailable) in one pass over the table:

```sql
SELECT
    COUNT(*) AS total_searches,
    COUNT(*) FILTER (WHERE "isValidSearch" = true) AS valid_searches,
    COUNT(*) FILTER (WHERE "isValidSearch" = false) AS invalid_searches,
    COUNT(*) FILTER (WHERE "isDomainAvailable" = true) AS available_domains,
    COUNT(*) FILTER (WHERE "isDomainAvailable" = false) AS unavailable_domains
FROM tbl_domain_search_log
```

But not one of these five newer routes carries any `@UseGuards(...)` decorator at all. They sit in the exact same controller, right below six routes that all require superadmin access, and yet anyone can call them. This is worth flagging with the same weight as the missing guards found in [05-landing-page-marketing-content.md](05-landing-page-marketing-content.md), because the pattern (an older, carefully guarded generation of endpoints sitting beside a newer, apparently forgotten generation) is common enough across this cluster to be worth watching for generally, not just in this one file.

Caching is, once again, applied selectively rather than uniformly, `getSearchTrend`, `getTldChart`, and `getSearchSummary` all carry `@UseInterceptors(CacheInterceptor)`, matching the root `app.module.ts` comment about aggregate, dashboard style endpoints, while `getSearchHistory` and `getTopTlds`, which are paginated, filterable list views rather than slow rollups, correctly do not.

## A separate cron keeps one dashboard chart fast without touching live traffic

`SearchLogRollupService`, registered in the same module, runs every fifteen minutes:

```ts
@Cron('*/15 * * * *', { name: 'refreshSearchLogDailyTldRollup', timeZone: 'UTC' })
async refreshOnSchedule(): Promise<void> {
    if (process.env.NODE_ENV === 'test') return;
    await this.refresh(); // REFRESH MATERIALIZED VIEW CONCURRENTLY mv_search_log_daily_tld_stats
}
```

This refreshes a Postgres materialized view rather than a plain table, and specifically uses the `CONCURRENTLY` variant, which lets reads against the existing view continue uninterrupted while the refresh runs, at the cost of needing a unique index on the view to be legal in Postgres. None of the controller code read in this file queries `mv_search_log_daily_tld_stats` directly, so its consumer either lives elsewhere in the codebase outside this cluster or is a rollup prepared ahead of a dashboard feature that has not been wired up to read it yet, either way, the pattern itself, keeping an expensive aggregate warm on a schedule so a dashboard never has to compute it live against a busy table, is worth recognizing on its own.

## `analytics`: the smallest, and most generic, thing in this entire cluster

The plain `analytics` folder is not a wrapper around Google Analytics, it does not read anything, and it has no relationship to `google-analytics` beyond a similar sounding name. It is a single write only endpoint:

```ts
@Post()
public async saveAnalyticsData(@Body() createAnalytics: CreateAnalyticsDto): Promise<Response> {
    return new Response('asdasd', await this.analyticsService.createAnalyticsLogs(createAnalytics));
}
```

`CreateAnalyticsDto` accepts a free-form `type` string (its own Swagger example lists `click`, `add_to_cart`, and `remove_from_cart`) and an arbitrary `payload` of any shape, and the service does nothing more than persist both straight into a `jsonb` column on `tbl_analytics`. There is no authentication guard on this route, no validation of what `type` values are allowed, and no visible downstream consumer of this data anywhere in this cluster, it is a generic catch-all event logger that some frontend code posts arbitrary interaction events into, most likely for later, manual analysis directly against the database rather than through any dashboard built in this codebase. The success message string, `'asdasd'`, left in the response constructor above, is a small but genuine, checkable piece of evidence that this endpoint shipped without a final pass, worth quoting exactly as it exists rather than smoothing it over, since it is the kind of detail a new engineer stumbling across this route in production would reasonably wonder about.
