# 07. Search Console Integration and the Unified Admin Dashboard

## The same authentication, deliberately reused rather than duplicated

`search-console` does not build its own connection to Google. `SearchConsoleModule` imports `GoogleAnalyticsModule` directly and reuses its exported `GoogleAnalyticsAuthServiceInterface` and `QuotaUsageTrackerServiceInterface` rather than standing up a second copy of either. This is spelled out explicitly in a comment on `google-analytics.module.ts` itself: `QuotaUsageTrackerService` lives in the Google Analytics module specifically so that both `Ga4Service` and `SearchConsoleModule`'s `GscService` "share the exact same GA4 and GSC quota counters rather than each tracking against a separate instance." One authenticated Google client, minted once from one service account, answers for both products, and one shared set of rate limit counters tracks usage across both, which matters because Google's own quota accounting does not care which internal service made the call, only that it came from the same underlying credentials.

## Calling the real Search Console API, and a documented misconception worth knowing

`GscService.getSearchAnalytics` calls `google.searchconsole('v1').searchanalytics.query`, the same Google product a marketer would use directly at search.google.com/search-console. A constant in this folder corrects a genuinely common misreading of Google's own documentation:

```ts
// Google enforces 25,000 as the hard maximum rows a single searchanalytics.query
// call can return, regardless of the rowLimit requested... The commonly cited
// "50,000 rows per day per site per search type" ceiling is the total volume
// reachable by paging through multiple calls with startRow, not a single call's
// rowLimit.
export const GSC_MAX_ROW_LIMIT = 25000;
export const GSC_TOTAL_ROW_CEILING_PER_DAY = 50000;
```

Anyone reading Google's Search Console documentation quickly and building a single request for fifty thousand rows would hit a real, confusing wall, this codebase already worked that out and encodes the correct behavior, cap any single request at twenty five thousand and page with `startRow` if more is genuinely needed. `GscReportQueryDto` enforces that same twenty five thousand ceiling with a `@Max(GSC_MAX_ROW_LIMIT)` validator, so a caller cannot even construct an invalid request. A `403` response from Google is translated into this codebase's own `ForbiddenException` naming the specific unverified site, rather than a raw Google error reaching the frontend.

## The sync job accounts for a delay that is specific to this API

```ts
const SYNC_WINDOW_DAYS = 7;
const PROCESSING_DELAY_WINDOW_DAYS = 2;

@Cron(CronExpression.EVERY_DAY_AT_4AM)
async syncAllProperties(): Promise<void> { ... }
```

Search Console's own documentation states that the most recent days of data are not yet finalized when queried. `GscSyncCronService` handles this by re-fetching a trailing seven day window every single night, rather than only yesterday, so a day that was still pending on an earlier sync gets its real numbers filled in once Google finishes processing it, instead of staying frozen at whatever partial value it had. Any row whose date falls within the last two days at sync time gets `isProcessingDelayed` set to `true` on the stored metric row, so the admin dashboard can label that day as pending rather than displaying a misleadingly final looking zero. This is a small but genuinely thoughtful detail, and a good example of a sync job that models the specific, real behavior of the API it is calling rather than treating every API the same way.

## The dashboard endpoint that combines both sources into one answer

`AnalyticsOverviewService.getOverview`, exposed at `GET /admin/analytics/overview` on the `AdminAnalyticsController` covered in [06-google-analytics-integration.md](06-google-analytics-integration.md), is the one place in this cluster where GA4 and Search Console data genuinely meet. A comment on the service explains a real, deliberate constraint, this system only ever bootstraps one enabled GA4 property and one enabled GSC site (see the bootstrap service in file 06), so this endpoint does not take a property id parameter at all, it just finds whichever single property of each type is currently enabled and reports both side by side:

```ts
return {
    ga4: ga4Row ? { users: ..., sessions: ..., pageViews: ..., engagementRate: ... } : null,
    gsc: gscRow ? { clicks: ..., impressions: ..., ctr: ..., position: ... } : null,
    lastSyncedAt,
};
```

`lastSyncedAt` is computed as the more recent of the two sources' own last successful sync timestamps, so the dashboard can tell an admin exactly how fresh the numbers on screen actually are, which matters for data that is only refreshed once a day by a cron job rather than fetched live on every page load.

## Why this lives in its own module rather than inside either integration

`AdminAnalyticsModule` exists specifically to avoid a circular dependency. `SearchConsoleModule` already imports `GoogleAnalyticsModule` (to reuse its auth and quota services, as described above), which means `GoogleAnalyticsModule` cannot import `SearchConsoleModule` back without creating a cycle, and `AdminAnalyticsController` genuinely needs both services at once to answer `/admin/analytics/overview` and its sibling routes. The fix is a third, small module that imports both existing modules and holds only the controller and the one service that needs both, sidestepping the cycle entirely without reaching for NestJS's `forwardRef()` escape hatch. This is a clean, general pattern worth recognizing, when module A needs module B and a third piece of code needs both A and B, that third piece belongs in its own small aggregating module rather than forcing a cycle onto either A or B.
