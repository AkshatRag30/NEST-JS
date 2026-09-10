# 06. Google Analytics Integration

## The auth pattern here is real, useful, and different from what "OAuth" usually means

It is easy to hear "this integrates with a real Google API" and assume it means the familiar consumer OAuth flow, a browser redirect to a Google consent screen, a callback URL, a refresh token stored per user. That is not what is happening here, and the actual pattern is arguably more useful to understand for backend work, because it is the one used whenever a server needs to talk to a Google API on its own behalf, with no human clicking "allow" anywhere in the loop. This integration authenticates as a Google Cloud service account, a machine identity created once in the Google Cloud console, using a private key that never expires on its own and is never tied to any one human user's login.

`GoogleAnalyticsAuthService.getAuthorizedClient` is the whole mechanism:

```ts
const client = new google.auth.JWT({
    email,                                  // GOOGLE_SERVICE_ACCOUNT_EMAIL
    key: this.normalizeAndValidatePrivateKey(rawKey),  // GOOGLE_PRIVATE_KEY
    scopes: ANALYTICS_AUTH_SCOPES,          // analytics.readonly, webmasters.readonly
});
```

The `googleapis` package's `google.auth.JWT` client signs its own short lived access tokens locally using that private key, and refreshes them automatically behind the scenes, so nothing in this codebase ever stores or refreshes an OAuth refresh token the way a consumer login flow would. The client is built once and cached in memory (`this.cachedClient`) for the lifetime of the process. The two scopes requested, `analytics.readonly` and `webmasters.readonly`, are also worth noticing, this integration can only ever read GA4 and Search Console data, it has no ability to change any setting in either product, which is a sensible, minimal permission grant for a backend that only needs to display numbers.

## The private key normalization code is worth reading in full

Whatever secret manager holds a multi line PEM private key, getting that exact value back out intact is a surprisingly common source of production incidents, and this codebase has clearly been bitten by it before. `normalizeAndValidatePrivateKey` handles, in order, a key whose real newlines were flattened into literal `\n` sequences by round tripping through AWS Secrets Manager's JSON encoding, a stray pair of wrapping quote characters left over from a console paste, Windows style `\r\n` line endings mixed into the body, a key with no `BEGIN`/`END` PEM markers at all because only the base64 body was copied, and a base64 body whose length is not a multiple of four, which the code explains is mathematically certain to mean the value was truncated:

```ts
const remainder = base64Body.length % 4;
if (remainder !== 0) {
    const missingChars = 4 - remainder;
    throw new InvalidGooglePrivateKeyException(
        `the key body is ${base64Body.length} characters long, which is not a multiple of 4 as valid base64 must be; ` +
        `it looks truncated by roughly ${missingChars} character(s), re-copy the complete key from Google Cloud including its final characters`
    );
}
```

Every error path here logs enough to diagnose the problem, the key's length, whether it contains the word `BEGIN`, and so on, while explicitly never logging the key material itself. This is a genuinely good example of defensive code written by someone who had already spent real time debugging an opaque `DECODER routines::unsupported` error with no way to inspect the secret directly, and it is worth remembering the next time a PEM key mysteriously fails to parse after passing through a secrets manager.

## Calling the actual GA4 API

`Ga4Service.getStandardReport` and `getRealtimeReport` both call `google.analyticsdata('v1beta')` from the `googleapis` package, Google's own typed client library, rather than hand rolling HTTP requests. One detail worth knowing, GA4's REST paths require the property identifier to be prefixed with `properties/`, and configuration in this system reasonably might only store the bare numeric id, so `toPropertyPath` normalizes that in exactly one place rather than trusting every caller to format it correctly. `retryWithBackoff` wraps the standard report call (but deliberately not the realtime call, since realtime data goes stale in seconds and retrying it defeats the point), and a `429` or `RESOURCE_EXHAUSTED` response from Google is caught and rethrown as this codebase's own `AnalyticsQuotaExceededException` rather than leaking Google's raw error shape up to the frontend.

## This service also protects Google's quota before Google has to

`QuotaUsageTrackerService.assertUnderLimit` is called before every outbound Google call, both for GA4 and, since the module is shared, for Search Console too:

```ts
const SOFT_LIMITS: Record<AnalyticsSource, { DAY: number; HOUR: number }> = {
    GA4: { DAY: 180000, HOUR: 36000 },
    GSC: { DAY: 40000, HOUR: 8000 },
};
```

A comment explains these numbers are deliberately kept around ten percent below Google's documented hard limits (200,000 Core tokens per day and 40,000 per hour for GA4, 50,000 rows per day per site per search type for GSC), so this system degrades with its own clear, internal `QuotaSoftLimitExceededException` well before Google itself would start rejecting calls outright. The counters live in `tbl_google_api_quota_usage`, one row per source, bucket type (day or hour), and bucket start time, and `incrementBucket` uses an atomic `INSERT ... ON CONFLICT ... DO UPDATE` rather than a read then write, specifically because multiple properties syncing at the same time could otherwise race on the same bucket and undercount real usage.

## Bootstrapping and syncing without needing an admin UI first

`AnalyticsPropertyBootstrapService.onModuleInit` reads the one configured `GA4_PROPERTY_ID` and `GSC_SITE_URL` straight out of the shared secrets bundle and registers them as `tbl_analytics_property` rows on every application boot, so the rest of this system, the sync jobs and the admin API, always has something to query without anyone needing to click through a setup screen first. It deliberately swallows its own errors rather than blocking startup, on the theory that a later, real attempt to sync or query will surface a clearer error at the moment it actually matters.

`Ga4SyncCronService` runs once a day at 3am UTC, and syncs each enabled property sequentially rather than in parallel, on purpose, a comment notes this is itself the concurrency limiter, keeping the system comfortably under Google's documented ten concurrent request ceiling. Each sync acquires a lock through `markRunning`, an `INSERT ... ON CONFLICT ... WHERE isRunning = false` that atomically handles three cases at once, no status row yet, a row that is not currently running, and a row that is already running, returning whether the lock was actually won. That same lock is reused by the manual resync endpoint on the admin controller (covered next in [07-search-console-and-unified-admin-dashboard.md](07-search-console-and-unified-admin-dashboard.md)), which is why a genuinely conflicting manual resync request gets a real `409 Conflict` back immediately rather than silently queuing behind a cron job that is already running.

## Where caching is confirmed to actually be applied

This is the answer to the specific question this cluster's notes were asked to settle. `AdminAnalyticsController`, defined in `admin-analytics.module.ts` (its own separate module, deliberately, to avoid a circular import between the GA4 and Search Console modules, explained in a comment on the module itself), applies caching exactly where the root `app.module.ts` comment about `CacheModule` says it should:

```ts
const CACHE_TTL_MS = 300000; // 5 minutes

@Get('overview')
@UseInterceptors(CacheInterceptor)
@CacheTTL(CACHE_TTL_MS)
async getOverview(@Query() query: GetAnalyticsOverviewDto) { ... }

@Get('ga4')
@UseInterceptors(CacheInterceptor)
@CacheTTL(CACHE_TTL_MS)
async getGa4Report(@Query() query: GetAdminAnalyticsReportDto) { ... }

@Get('gsc')
@UseInterceptors(CacheInterceptor)
@CacheTTL(CACHE_TTL_MS)
async getGscReport(@Query() query: GetAdminAnalyticsReportDto) { ... }
```

The realtime endpoint, `ga4/realtime`, is deliberately left uncached, with a comment explaining that real time data is stale within seconds and the frontend's own polling interval, not a server side cache, is what keeps it from hammering Google's realtime quota. The manual resync endpoint is not cached either, it is throttled instead, to once per minute per caller. Every one of these choices matches its own stated intent, this is the clearest, most direct confirmation anywhere in this cluster that the caching convention described in the root module is genuinely followed, at least here.
