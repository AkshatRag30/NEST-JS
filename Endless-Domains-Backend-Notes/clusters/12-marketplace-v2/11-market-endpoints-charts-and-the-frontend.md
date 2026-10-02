# 11. Market endpoints, the chart series, and how the frontend should use them

## Where we are

The previous note, [`10-opensea-market-data-pipeline.md`](10-opensea-market-data-pipeline.md), followed data from OpenSea and CoinGecko through the background jobs into six in memory cache keys. This note sits on the other side of that cache. It covers the four HTTP routes the module exposes, the exact JSON each one returns, how the chart series is bucketed (including the brand new average sale price chart from commit `ebb36cd0`), how a React dashboard should fetch and draw all of it without being fooled by placeholders and partial hours, and finally every spec in the module, file by file.

## The four routes at a glance

The app sets `app.setGlobalPrefix('api/v1')` in `main.ts`, so every controller path below is prefixed with `/api/v1`.

| Method and path | Guards | Wrapper | Reads | Before first run |
|---|---|---|---|---|
| `GET /api/v1/market/stats` | none, public | raw JSON | `market:stats` | 503 |
| `GET /api/v1/market/series` | none, public | raw JSON | `market:series` | 503 |
| `GET /api/v1/market/activity` | none, public | raw JSON | `market:activity` | 503 |
| `GET /api/v1/marketplacev2/market-data/_health` | `AccessTokenGuard`, `AdminTokenGuard` | `Response` | config, status Map, client snapshot, cache ages | always 200 |

None of the routes take a query string, a body or a path parameter, so there are no DTO classes and nothing for the global `ValidationPipe` to do. That is a deliberate shape: the windows (24h, 7d, 30d) are all returned at once, and the frontend picks the one it wants to show, which keeps the backend a pure cache lookup with a single key per route.

## The public controller and why it breaks the house rule

```ts
// src/components/marketplacev2/market-data/market.controller.ts
@Controller('market')
export class MarketController {
    constructor(private readonly marketReadService: MarketReadService) {}

    @Get('stats')
    async stats(): Promise<MarketStatsResponse> {
        return this.marketReadService.getStats();
    }

    /** Chart series for the 24h, 7d and 30d windows: volume, sales and new listings per time bucket. A cache read, like the others. */
    @Get('series')
    async series(): Promise<MarketSeriesResponse> {
        return this.marketReadService.getSeries();
    }

    /** Newest 40 sales and listings. The page polls this every 30 seconds, so it must stay a cache read. */
    @Get('activity')
    async activity(): Promise<MarketActivityResponse> {
        return this.marketReadService.getActivity();
    }
}
```

Every other controller in this codebase returns `new Response(message, data)`, the `{ success, statusCode, message, result }` envelope described in the root CLAUDE.md. This one does not, and the class comment explains why: these shapes are a "frozen frontend contract" that has to survive a later change of data source. If one day the numbers come from a different provider or our own indexer, the landing page should not notice. Returning the payload raw means the frontend reads `data.industry['24h'].volumeUsd`, not `data.result.industry...`. If you are writing the fetch code, this is the first thing to get right, and the most likely thing to trip a teammate who copies a fetch helper from elsewhere in the app that unwraps `.result`.

The read service is the whole backend for these routes:

```ts
// src/components/marketplacev2/market-data/market-read.service.ts
async getStats(): Promise<MarketStatsResponse> {
    const stats = await this.cache.get<MarketStatsResponse>(MARKET_CACHE_KEYS.STATS);
    if (!stats) {
        throw new ServiceUnavailableException('Market stats are not ready yet, the first data run has not finished');
    }
    return stats;
}
```

`getSeries` and `getActivity` are identical apart from the key and the message (`'Market chart series are not ready yet, the first data run has not finished'` and `'Market activity is not ready yet, the first data run has not finished'`). The 503 rather than an empty shape is a thoughtful choice: an object full of zeros would render as "the domain market is dead today", which is a lie, while a 503 tells the frontend to show a loading or "calculating" state. Because there is no custom exception filter, the 503 body is Nest's default: `{ "statusCode": 503, "message": "Market stats are not ready yet, the first data run has not finished", "error": "Service Unavailable" }`.

There is no `CacheInterceptor`, no `Cache-Control` or `ETag` header, and no throttling on these routes. `ThrottlerModule.forRoot([{ ttl: 60000, limit: 60 }])` is registered in `app.module.ts:116` but no `APP_GUARD` applies it, so a bot can hammer these routes freely. Since each request is a single in memory map lookup, that is cheap, but a CDN cache header of 30 to 60 seconds would cost nothing and take load off Node. CORS still applies from `main.ts`, so the landing page has to be on one of the whitelisted origins.

## `GET /api/v1/market/stats`, the four tiles

The type lives in `dto/market-stats.response.ts`:

```ts
// src/components/marketplacev2/market-data/dto/market-stats.response.ts
export interface MarketStatsWindowBlock {
    volumeUsd: number;
    sales: number;
    newListings: number;
    avgSaleUsd: number;
    /** True when at least one contributing collection hit the listings page cap: show "20k+" instead of an exact number. */
    listingsTruncated: boolean;
}

export type MarketStatsWindows = Record<MarketWindow, MarketStatsWindowBlock>;

export interface MarketStatsResponse {
    generatedAt: string;
    source: string;
    industry: MarketStatsWindows;
    byService: Record<string, MarketStatsWindows>;
    coverage: { services: string[]; chains: string[] };
    listingsReady: boolean;
}
```

Field by field. `generatedAt` is the ISO time `recompose()` ran, for "updated 2 minutes ago". `source` is the fixed `MARKET_STATS_SOURCE` string explaining that USD is at the current CoinGecko price. `industry` holds one block per window, `'24h'`, `'7d'` and `'30d'` (the `MarketWindow` union from `MARKET_WINDOWS`). `byService` has the same block structure keyed by service label, and only services that produced data appear, so do not assume all four keys are always there. Inside a block, `volumeUsd` is rounded to two decimals, `sales` and `newListings` are whole numbers, `avgSaleUsd` is total volume over total sales rounded to two decimals and is `0` when there were no sales, and `listingsTruncated` is true when the count is a floor for that specific window (ENS is typically truncated only for 30d). `coverage.services` and `coverage.chains` are the sorted distinct values of collections that produced data, so the untracked bsc collections never appear. `listingsReady` is false until a listings run has produced at least one collection's counts after a restart; while it is false every `newListings` is a placeholder zero.

Here is a realistic payload, written as a typed object so the shape is checked:

```ts
// example: what GET /api/v1/market/stats returns (values illustrative)
const example: MarketStatsResponse = {
    generatedAt: '2026-10-02T10:30:04.512Z',
    source: 'OpenSea API, USD via CoinGecko at snapshot time (windowed volumes are converted at the current price)',
    industry: {
        '24h': { volumeUsd: 48210.37, sales: 41, newListings: 1904, avgSaleUsd: 1175.86, listingsTruncated: false },
        '7d': { volumeUsd: 301877.1, sales: 233, newListings: 12650, avgSaleUsd: 1295.61, listingsTruncated: false },
        '30d': { volumeUsd: 1188420.55, sales: 902, newListings: 24311, avgSaleUsd: 1317.54, listingsTruncated: true }
    },
    byService: {
        ENS: { '24h': { volumeUsd: 46100.12, sales: 22, newListings: 702, avgSaleUsd: 2095.46, listingsTruncated: false }, '7d': { /* ... */ } as any, '30d': { /* ... */ } as any },
        'Unstoppable Domains': { /* same three windows */ } as any
    },
    coverage: { services: ['ENS', 'Freename', 'SPACE ID', 'Unstoppable Domains'], chains: ['arbitrum', 'base', 'ethereum', 'matic'] },
    listingsReady: true
};
```

## `GET /api/v1/market/activity`, the live feed

```ts
// src/components/marketplacev2/market-data/dto/market-activity.response.ts
export interface MarketActivityItem {
    type: 'sale' | 'listing';
    service: string;
    chain: string;
    name: string;
    image: string | null;
    price: number | null;
    symbol: string | null;
    priceUsd: number | null;
    timestamp: number;
    tx: string | null;
}

export interface MarketActivityResponse {
    generatedAt: string;
    items: MarketActivityItem[];
}
```

`items` holds at most 40 entries (`ACTIVITY_KEEP`), sorted newest first, with the newest 10 sales always included even if forty listings are newer. `type` comes from which OpenSea query produced the item. `chain` is our registry spelling (`matic`, `bsc`), not OpenSea's per event spelling. `name` falls back to `#<first 10 chars of token id>` and then `Unnamed`. `image` is OpenSea's `image_url` or null. `price` is the amount in the payment token as a JavaScript number (for a listing, the asking price), `symbol` is the token symbol as OpenSea gave it, `priceUsd` is rounded to four decimals and is null when there is no payment or the token is not in the price table. `timestamp` is unix seconds, not milliseconds, so multiply by 1000 before `new Date()`. `tx` is the sale's transaction hash and always null for listings, so a "view on explorer" link only exists for sales, and you need the chain to build the explorer URL. The controller comment says the page polls this every 30 seconds while the job refreshes every 60, so on average every other poll returns the same `generatedAt`.

## `GET /api/v1/market/series`, the chart data

```ts
// src/components/marketplacev2/market-data/dto/market-series.response.ts
export interface MarketSeriesWindowBlock {
    /** 3600 for 24h, 86400 for 7d and 30d. */
    bucketSeconds: number;
    /** Unix seconds at the start of the first (oldest) bucket. Bucket i starts at startUnix + i * bucketSeconds. */
    startUnix: number;
    /** USD, converted per sale at the run's prices. */
    volumeUsd: number[];
    sales: number[];
    newListings: number[];
    /**
     * Average sale price in USD per bucket: that bucket's volume divided by its sales. A bucket with no sales has
     * no average, so it is `null` (not 0, which would draw a false dip to zero). Draw it with the gaps bridged.
     */
    avgSaleUsd: Array<number | null>;
    /** True when a collection hit the listings page cap and the oldest buckets of newListings are understated. */
    listingsTruncated: boolean;
}

export interface MarketSeriesResponse {
    generatedAt: string;
    listingsReady: boolean;
    windows: Record<MarketWindow, MarketSeriesWindowBlock>;
}
```

Each window gives you four parallel lists of equal length, oldest first, plus enough timing information to compute the x axis yourself. The lengths never change: 24 for `'24h'`, 7 for `'7d'`, 30 for `'30d'`. Empty buckets are `0` in the three count lists and `null` in `avgSaleUsd`. Note that the series is industry wide only; there is no per service breakdown in the series, unlike the stats.

## How the series is built, step by step

### The raw material: hourly bins

Both counters fill an hourly array in the same pass that counts windows. The constants are:

```ts
// src/components/marketplacev2/market-data/constants/market-data.constants.ts
export const HOUR_SECONDS = 3600;
export const HOURLY_BINS = 30 * 24 + 2;
export const SERIES_WINDOWS: Readonly<Record<MarketWindow, { buckets: number; bucketSeconds: number }>> = {
    '24h': { buckets: 24, bucketSeconds: 3600 },
    '7d': { buckets: 7, bucketSeconds: 86400 },
    '30d': { buckets: 30, bucketSeconds: 86400 }
};
```

A sale with `event_timestamp = at` goes into bin `Math.floor(at / 3600) - baseHour`, where `baseHour = Math.floor((now - 30 days) / 3600)` at the moment that counter ran. So bin `i` is the absolute unix hour `baseHour + i`. Sales bins carry `sales[]` and `volumeUsd[]` (`HourlySalesBins` in `stats/stats.types.ts`); listing bins carry `counts[]`. The 722 bins are 30 days plus two hours of slack for the partial hours at both edges. Bins are stored by absolute hour so a sales result counted at 10:00 and a listings result counted at 09:07 can be added together without any shifting, which is the key idea that makes the whole builder simple.

### The builder

```ts
// src/components/marketplacev2/market-data/series/series-builder.ts
export function buildSeries(input: { results: CollectionStatsResult[]; listings: ListingsCachePayload | undefined; now: Date }): MarketSeriesResponse {
    const nowSeconds = Math.floor(input.now.getTime() / 1000);
    const endHour = Math.floor(nowSeconds / HOUR_SECONDS) + 1;

    const windows = {} as MarketSeriesResponse['windows'];
    for (const window of MARKET_WINDOWS) {
        const { buckets, bucketSeconds } = SERIES_WINDOWS[window];
        const hoursPerBucket = bucketSeconds / HOUR_SECONDS;
        const startHour = endHour - buckets * hoursPerBucket;

        const volumeUsd = new Array<number>(buckets).fill(0);
        const sales = new Array<number>(buckets).fill(0);
        const newListings = new Array<number>(buckets).fill(0);
        let listingsTruncated = false;

        for (const result of input.results) {
            const salesBins = result.hourly;
            const counted = input.listings?.slugs?.[result.slug];
            const listingBins = counted?.hourly;

            for (let bucket = 0; bucket < buckets; bucket++) {
                for (let h = 0; h < hoursPerBucket; h++) {
                    const hour = startHour + bucket * hoursPerBucket + h;
                    if (salesBins) {
                        const index = hour - salesBins.baseHour;
                        if (index >= 0 && index < HOURLY_BINS) {
                            sales[bucket] += salesBins.sales[index] ?? 0;
                            volumeUsd[bucket] += salesBins.volumeUsd[index] ?? 0;
                        }
                    }
                    if (listingBins) {
                        const index = hour - listingBins.baseHour;
                        if (index >= 0 && index < HOURLY_BINS) {
                            newListings[bucket] += listingBins.counts[index] ?? 0;
                        }
                    }
                }
            }

            if (counted) {
                const truncated = counted.truncatedWindows ? counted.truncatedWindows[window] === true : counted.truncated === true;
                listingsTruncated = listingsTruncated || truncated;
            }
        }

        // Average sale per bucket from the unrounded volume. No sales in a bucket means no average (null), not 0.
        const avgSaleUsd = volumeUsd.map((volume, i) => (sales[i] > 0 ? round2(volume / sales[i]) : null));

        const block: MarketSeriesWindowBlock = {
            bucketSeconds,
            startUnix: startHour * HOUR_SECONDS,
            volumeUsd: volumeUsd.map(round2),
            sales,
            newListings,
            avgSaleUsd,
            listingsTruncated
        };
        windows[window] = block;
    }

    return {
        generatedAt: input.now.toISOString(),
        listingsReady: input.listings !== undefined && Object.keys(input.listings.slugs ?? {}).length > 0,
        windows
    };
}
```

It is a pure function, no I/O, with `now` passed in, which is exactly what makes it so testable. `recompose()` in `stats/stats.job.ts` calls it with the same `results` and `listings` and the same `now` it uses for the stats payload, so the two always describe the same collections at the same moment. Only collections present in `results` contribute, including their listings; a listings entry for a slug that is not in the sales results is ignored, mirroring the stats totals.

### A worked example with real numbers

The spec fixes `NOW = 2026-10-02T10:30:00Z`, so let us use that. `nowSeconds` is 1790850600, `Math.floor(nowSeconds / 3600)` is the hour starting 10:00 UTC, and `endHour` is the next hour, 11:00 UTC. The window therefore ends at the end of the hour in progress, and its last bucket is "now".

| Window | `buckets` x hours | `startHour` | `startUnix` | First bucket | Last bucket |
|---|---|---|---|---|---|
| `24h` | 24 x 1 | `endHour - 24` | 1790852400 | 1 Oct 2026 11:00 to 12:00 UTC | 2 Oct 2026 10:00 to 11:00 UTC (in progress) |
| `7d` | 7 x 24 | `endHour - 168` | 1790334000 | 25 Sep 11:00 to 26 Sep 11:00 UTC | 1 Oct 11:00 to 2 Oct 11:00 UTC |
| `30d` | 30 x 24 | `endHour - 720` | 1788346800 | 2 Sep 11:00 to 3 Sep 11:00 UTC | 1 Oct 11:00 to 2 Oct 11:00 UTC |

So bucket `i` of any window covers `[startUnix + i * bucketSeconds, startUnix + (i + 1) * bucketSeconds)`. If the sales run happened at 10:30, its `baseHour` is the hour starting 10:00 UTC on 2 September, so the 30d series reads bins 1 through 720 and never touches bin 0 (the partial first hour, which only holds sales after 10:30 on 2 September) or bin 721.

### How each chart is bucketed

The volume chart is the sum of every collection's `volumeUsd` bins inside each bucket, converted per sale at the run's prices and rounded to two decimals only at the end. The sales chart is the plain sum of sale counts, including sales whose token had no USD price. The new listings chart is the sum of listing counts from the latest listings run. The average sale price chart, new in `ebb36cd0`, is that bucket's unrounded volume divided by that bucket's sales, rounded to two decimals, or `null` if the bucket had no sales. Because both the numerator and the denominator are summed across every collection and every hour first, it is a true weighted average, never an average of averages; the spec "a daily bucket is its total volume over its total sales" proves one hour with 1 sale of $1,000 and another with 9 sales totalling $90 gives a day average of $109, not the $505 a naive mean would give.

## The honest caveats for anyone drawing these charts

The daily buckets are not calendar days. They are 24 hour blocks ending at the end of the current UTC hour, so at 10:30 UTC each "day" runs 11:00 to 11:00 UTC (`series-builder.ts:30` and `:36`). For a team and audience in India, UTC plus 5:30, the hourly buckets start at half past the hour in local time and each daily bucket runs from 16:30 to 16:30 IST. If the chart labels a daily bar "1 Oct", it is lying a little: that bar is really "the 24 hours ending 16:30 IST on 2 Oct". Label daily bars by their end time or as "n days ago", and format hourly ticks from `startUnix` in the viewer's time zone. The boundaries also shift every hour, so a bar's value changes as hours roll in and out of it even with no new sales.

The rightmost buckets are partial in two different ways. The last bucket is the hour (or 24 hours) in progress, so it is naturally low until that period ends. On top of that, the series is only rebuilt when a recompose runs (every 15 minutes, plus after the hourly listings run), and the listing bins come from a run that started at minute 7 and took several minutes. At 11:00 the stats recompose builds a series whose current hour has no listing data at all and whose previous hour has only listings up to about 10:10, so the new listings line dips sharply at its right edge until the 11:07 run completes. Sales have the milder version: the last bucket only contains sales up to the moment the stats run read them. A chart that shows a falling last point is mostly showing timing, not market behaviour, so consider drawing the final bucket dashed or lighter.

The series and the tiles will not match exactly. The tiles use an exact rolling window from the run time (now minus 24 hours to the second), while the series is aligned to whole hours, so summing the 24h series can differ from the 24h tile by the events in the partial hours at the edges. The builder's own comment says the headline number should come from the stats and the shape from the series. Do not sum the series to produce a headline.

`listingsTruncated` is a single flag per window and does not say which buckets are understated. When it is true for 30d (ENS normally hits the 400 page cap before reaching 30 days back), the oldest few daily buckets of `newListings` are too low, which creates a fake rising trend on the 30d listings chart. Annotate it, or fade the oldest buckets.

Stale collections drift out of the chart. A collection whose sales or listings counting failed keeps its old result (see the previous note). Its bins are placed by absolute hour, so as hours pass, its recent buckets read as zero in the chart while its old 24h tile number stays frozen. A listings entry older than about an hour also contributes nothing to the current hour, because the right edge index exceeds its stored bins.

Average sale price reads low when some sales were paid in tokens outside the price table, because those sales count in the denominator but add nothing to the numerator. And the stats tile uses `0` for "no sales" while the series uses `null`, so the same idea has two encodings; in the tile, treat `sales === 0` as "no average" rather than drawing "$0.00".

Every USD figure in all three payloads is at today's price. A 30d volume chart therefore shifts up and down with ETH's price across every bar each time the stats job runs, even for days long past.

## Consuming it from React

The cleanest approach is three small hooks, each polling its own endpoint, each treating 503 as "not ready" rather than as an error. Here is a sketch with TanStack Query (the same idea works with SWR or a plain `useEffect`), assuming the backend types are copied into the frontend or shared through a package:

```ts
// frontend example: src/features/market/useMarketData.ts
import { useQuery } from '@tanstack/react-query';

const API = import.meta.env.VITE_API_URL; // e.g. https://api.endlessdomains.io/api/v1

class NotReadyError extends Error {}

async function getRaw<T>(path: string): Promise<T> {
    const res = await fetch(`${API}${path}`, { credentials: 'include' });
    if (res.status === 503) throw new NotReadyError('not ready');
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    // Raw JSON: these routes do NOT use the { success, result } wrapper.
    return (await res.json()) as T;
}

const retryNotReady = (count: number, err: unknown) => err instanceof NotReadyError && count < 20;

export const useMarketStats = () =>
    useQuery({ queryKey: ['market', 'stats'], queryFn: () => getRaw<MarketStatsResponse>('/market/stats'), refetchInterval: 60_000, retry: retryNotReady, retryDelay: 5_000 });

export const useMarketSeries = () =>
    useQuery({ queryKey: ['market', 'series'], queryFn: () => getRaw<MarketSeriesResponse>('/market/series'), refetchInterval: 60_000, retry: retryNotReady, retryDelay: 5_000 });

export const useMarketActivity = () =>
    useQuery({ queryKey: ['market', 'activity'], queryFn: () => getRaw<MarketActivityResponse>('/market/activity'), refetchInterval: 30_000, retry: retryNotReady, retryDelay: 5_000 });
```

Polling more often than once a minute for stats or series buys nothing, since the data changes at most every 15 minutes; the 30 second activity poll matches the controller comment. Turning a series window into chart points is a few lines, and this is where the `null` handling for the average chart lives:

```ts
// frontend example: src/features/market/toChartPoints.ts
export interface SeriesPoint {
    start: Date;
    end: Date;
    volumeUsd: number;
    sales: number;
    newListings: number | null;
    avgSaleUsd: number | null;
    partial: boolean;
}

export function toChartPoints(block: MarketSeriesWindowBlock, listingsReady: boolean): SeriesPoint[] {
    return block.volumeUsd.map((volumeUsd, i) => {
        const startSec = block.startUnix + i * block.bucketSeconds;
        return {
            start: new Date(startSec * 1000),
            end: new Date((startSec + block.bucketSeconds) * 1000),
            volumeUsd,
            sales: block.sales[i],
            // Placeholder zeros are not real zeros: hide the listings line until the first listings run.
            newListings: listingsReady ? block.newListings[i] : null,
            avgSaleUsd: block.avgSaleUsd[i],
            partial: i === block.volumeUsd.length - 1
        };
    });
}
```

With Recharts, draw the average sale price as a `<Line dataKey="avgSaleUsd" connectNulls />` so gaps between sales are bridged rather than plunging to zero, which is precisely why the backend sends `null` and not `0`. Draw volume as bars and sales and listings as lines or bars on a second axis, since they are counts and volume is dollars. Format the x axis from `start` with `Intl.DateTimeFormat` in the viewer's time zone, using hours for `'24h'` and day labels based on `end` for `'7d'` and `'30d'`. Mark the `partial` point visually. When `block.listingsTruncated` is true, add a small "older listings undercounted" note. For the tiles, read `industry[window]` or `byService[service][window]`, show "calculating" for `newListings` when `listingsReady` is false, "20k+" style text when `listingsTruncated` is true, "no sales" when `sales` is zero instead of "$0.00" average, and use `generatedAt` with a relative time formatter for "updated 3 minutes ago". If `generatedAt` is more than, say, an hour old, it is worth dimming the tiles, because a stalled job keeps serving old numbers that look perfectly normal. For the activity feed, use a stable React key such as `${item.tx ?? item.name}-${item.timestamp}-${item.type}`, multiply `timestamp` by 1000, show `price symbol` with `priceUsd` as secondary text when present, and only render an explorer link when `tx` is not null.

## The health route in detail

```ts
// src/components/marketplacev2/market-data/market-data.health.controller.ts
@Controller('marketplacev2/market-data')
export class MarketDataHealthController {
    constructor(private readonly healthService: MarketDataHealthService) {}

    @ApiBearerAuth('defaultBearerAuth')
    @Get('_health')
    @UseGuards(AccessTokenGuard, AdminTokenGuard)
    async health(): Promise<Response> {
        return new Response('marketplacev2 market data health check', await this.healthService.getHealth());
    }
}
```

To call it you need a valid user access token as a Bearer header and the static admin token in an `admin-token` header. The body is `{ message: 'marketplacev2 market data health check', result: MarketDataHealth }`; since nothing calls `setSuccess` or `setStatusCode`, the `success` and `statusCode` fields of the wrapper are undefined and drop out of the JSON. The `result` is described in the previous note: `module`, `status`, `enabled`, `collections`, `healthy`, `jobs` (four `MarketJobStatus` objects with `lastRunAt`, `lastSuccessAt`, `lastError`, `lastDurationMs`, `runs`, `failures`, `skipped`, `healthy`), `rateLimit`, `cache` ages in seconds for stats, listings and activity, and `untracked`. An admin dashboard tile showing `healthy`, each job's `lastSuccessAt`, and the three cache ages is the fastest way to spot a stalled pipeline.

## Every spec in the module, file by file

The module has 22 spec files and 333 `it` cases. Most carry a `TC` number tying them to the master plan's test case list (TC1 is Sprint 1 config, TC2 the client, TC3 pricing, TC4 slugs and the old stats, TC5 listings, TC6 aggregation and the stats endpoint, TC7 activity, TC8 scheduler, health and acceptance). They all use plain Jest with hand built mocks or a fake clock; only `market.controller.spec.ts` boots a Nest testing app with supertest, and the cache specs use a real `cache-manager` instance from `createCache`.

### `config/market-data-config.service.spec.ts`

| Test | What it proves |
|---|---|
| TC1.1 valid secret, enabled, nine collections | With every key set, `enabled` is true and all nine required pairs resolve. |
| TC1.2 disabled with no API key | `MARKET_DATA_ENABLED=false` boots fine and `openSeaApiKey` is null. |
| TC1.2b missing flag means disabled | An absent `MARKET_DATA_ENABLED` is treated as false. |
| TC1.3 enabled without key fails | Throws, and the message names `OPENSEA_API_KEY` but contains no value. |
| TC1.3b blank key is missing | A whitespace only key counts as absent. |
| TC1.4 invalid address skipped | Warning names the key, never the bad value. |
| TC1.4b wrong checksum rejected | A mixed case address with a bad checksum fails `isAddress`. |
| TC1.5 missing key skipped | One missing key warns and the rest still resolve. |
| TC1.6 optional UD Ethereum pair | Included only when `MARKET_UD_ETH_CONTRACT` is present. |
| TC1.6b optional absent is silent | No warning for a missing optional key. |
| TC1.12 registry hygiene | Only the five allowed chains, unique contract keys, no hardcoded addresses. |
| never logs the API key | Nothing logged contains the key even when fully configured. |

### `market-data.cache.service.spec.ts`

| Test | What it proves |
|---|---|
| TC1.7 no implicit TTL | Missing, negative or `NaN` TTL rejects with "explicit" and writes nothing. |
| TC1.8 no expiry outlives default | With a 50 ms global default, a `CACHE_NO_EXPIRY` value survives 200 ms while a 50 ms value expires. |
| getWithAge | Reports a non negative age and undefined for a missing key. |
| second set overwrites | Last write wins. |

### `market-data.module.spec.ts`

| Test | What it proves |
|---|---|
| TC1.1 module compiles | All providers resolve and `onModuleInit` runs cleanly when disabled with no key. |
| TC1.3 boot fails | Enabled without an API key makes app init reject. |

### `market-data.health.controller.spec.ts`

| Test | What it proves |
|---|---|
| TC1.9 Response wrapper | Returns a `Response` instance whose JSON contains no "key", "secret" or address. |
| behind AccessTokenGuard | The guard metadata includes `AccessTokenGuard` (it does not check `AdminTokenGuard`). |
| mounted path | Controller path `marketplacev2/market-data`, route `_health`. |

### `opensea/opensea-request-queue.spec.ts`

| Test | What it proves |
|---|---|
| TC2.2 serial with 300 ms gap | Tasks never overlap and starts are at least 300 ms apart on the fake clock. |
| TC2.3 pause holds the next task | Nothing starts before `pausedUntil`. |
| pause only extends | A shorter `pauseUntil` is ignored. |
| pause extended while waiting | The loop in `acquireSlot` honours an extension made mid sleep. |
| TC2.10 priority | High runs ahead of waiting normal tasks, FIFO inside each priority. |
| no interruption | A running task finishes before a newly queued high task starts. |
| failing task | Rejects its own promise and the lane continues. |
| retry obeys gap | A running task calling `acquireSlot` still waits the 300 ms. |

### `opensea/opensea.client.spec.ts`

| Test | What it proves |
|---|---|
| TC2.1 headers, base URL, paths | Every method sends `x-api-key` to the right `/api/v2` path. |
| returns body | Response data passes through untouched. |
| url encodes slug | Special characters in a slug are encoded. |
| getEvents params | Sends `event_type`, `after`, `next`, and clamps `limit` to 50. |
| no throw on status, timeout set | `validateStatus` always true and a timeout is passed. |
| refuses when disabled | Zero HTTP calls and `OpenSeaNotConfiguredError` when disabled or keyless. |
| TC2.2 client level gap | Two calls start 300 ms apart and never overlap. |
| TC2.3 remaining below 5 pauses | Pause set to `reset * 1000` and the next call waits for it. |
| TC2.4 exactly 5 does not pause | Threshold is strictly below 5. |
| far future reset capped | A reset a day away is capped at now plus 10 minutes. |
| past reset no wait | A reset in the past causes no delay. |
| TC2.5 `Retry-After` of 2 | Waits 2 s, retries once, returns the 200 body. |
| retry is same request | Same URL and params on the retry. |
| TC2.6 five 429s | Throws `OpenSeaRateLimitError` after exactly five attempts. |
| TC2.7 missing `Retry-After` | Uses the 30 s fallback wait. |
| HTTP date in `Retry-After` | Also uses the fallback. |
| retries honour gap | Even with `Retry-After` 0, the 300 ms gap applies. |
| 429 never sets reset pause | Only `Retry-After` governs a 429. |
| lane held during 429 wait | No other request is sent while waiting. |
| TC2.8 500 not retried | One call, typed error. |
| TC2.9 404 typed | `OpenSeaRequestError` carries status 404 and the path. |
| transport failure | Typed error with null status, no retry. |
| next request continues | One failure does not block the next queued call. |
| TC2.10 priority passthrough | Requested priority reaches the queue, normal by default. |
| TC2.11 key never leaks | Even an axios style error carrying config headers produces no key in errors or logs. |
| rate limit snapshot and log | Headers recorded and logged by `logRateLimitSummary`. |
| no headers | Snapshot stays empty and nothing pauses. |
| TC2.12 one URL file | `api.opensea.io` appears in exactly one non test file. |

### `pricing/market-price.service.spec.ts`

| Test | What it proves |
|---|---|
| TC3.1 one GET, three ids | Exact URL with `ids=ethereum,binancecoin,polygon-ecosystem-token&vs_currencies=usd`. |
| snapshot immutable | The frozen snapshot cannot be mutated. |
| TC3.2 ETH and WETH | Same price, case insensitive. |
| BNB and WBNB | Same price. |
| TC3.3 POL family | POL, MATIC, WPOL, WMATIC share one price. |
| TC3.4 stablecoins | Exactly 1, no fetched price needed. |
| TC3.5 unknown symbol | Returns null with one warning naming it. |
| TC3.6 logged once per run | Three uses, one log line. |
| new symbol or new run logs again | Per snapshot dedupe via the WeakMap. |
| missing symbol | Treated as unknown, no crash. |
| numeric strings | Accepted; non numbers return null. |
| TC3.7 HTTP failure | Throws, no snapshot. |
| TC3.7 missing price | Throws, no partial snapshot. |
| invalid prices | Empty body, zero, negative and non numeric all reject. |
| TC3.8 demo key header | Sent only when configured. |
| demo key not in error | Thrown error never contains it. |
| TC3.9 native token per chain | ETH for ethereum, base, arbitrum; POL for matic; BNB for bsc. |
| native tokens priceable | Fallback never yields an unknown symbol. |
| table maps to fetched ids | Every mapped id is one of the three fetched. |

### `stats/slug-resolver.service.spec.ts`

| Test | What it proves |
|---|---|
| TC4.1 dedupe | Two pairs with one slug collapse, the first owning service and chain. |
| TC4.2 cached within 24 h | No requests on a second call, same output. |
| written with no expiry | Slug cache uses TTL 0. |
| fully cached writes nothing | No cache write when nothing changed. |
| resolved again after 24 h | Expired pairs are fetched again. |
| TC4.3 404 pair | Logged with label, chain and address, others still resolve. |
| no collection in response | Skipped with a warning and not cached. |
| TC4.4 refresh failure | Keeps the previously resolved slug. |
| successful refresh | Replaces the old slug. |
| all fail | Returns an empty list instead of throwing. |
| overlapping calls | Share one set of requests via `inFlight`. |
| untracked filtered | Left out of the result, back after the recheck window. |
| untracked stays cached | Costs no new resolve when it returns. |
| address casing | Cache key is lowercased. |

### `stats/collection-stats.service.spec.ts` (tests the retired service)

| Test | What it proves |
|---|---|
| TC4.5 WETH volume | Converted at the ETH price per window. |
| TC4.6 symbol fallback | matic falls back to POL, bsc to BNB. |
| TC4.7 interval mapping | `one_day`, `seven_day`, `thirty_day` map to the three windows. |
| matched by name | Interval order does not matter. |
| TC4.8 missing interval | Zero for that window, one log line. |
| empty body | Zero everywhere, not an error. |
| TC4.9 unknown symbol | Null USD, logged, sales still count. |
| zero volume unknown currency | Zero dollars, not null. |
| numeric strings and junk | Parsed or zeroed safely. |
| TC4.10 floor | Saved per slug, never summed. |
| failing slug | Logged, others return. |
| TC4.11 all fail | Throws so the caller skips its write. |
| empty slug list | Throws. |
| rate limit log | Logged once per run. |
| TC4.12 only via client | No direct `httpClient` use. |

### `sales/sales-counter.service.spec.ts`

| Test | What it proves |
|---|---|
| one pass bucketing | Sales fall into 24h, 7d, 30d and convert at run prices. |
| inclusive boundary | Exactly 24 h ago counts, one second older does not. |
| WETH and USDT sales | The very sales the stats endpoint missed now count. |
| unknown token | Counts as a sale, adds no volume, reported. |
| no payment | Counts as a sale with no volume. |
| no timestamp | Not counted, logged. |
| pagination | Follows the cursor with `event_type=sale`, `after` now minus 30 days, `limit` 50. |
| normal priority | No priority option passed. |
| empty result | Zeros, not an error. |
| result shape | Same as the old stats shape plus `hourly`, floor null. |
| failing collection | Logged, others count. |
| keeps previous figures | A failed collection reuses its last good result. |
| no earlier figures | A failed collection is simply absent. |
| all fail fresh | Throws even if earlier figures exist. |
| empty slug list | Throws. |
| repeated cursor | Treated as failure, no endless loop. |
| page cap | Logged as a floor, figures still returned. |
| rate limit log | Once per run. |
| 404 untracked | Marked, logged once, others continue. |
| 500 is not untracked | Still a failure. |
| hourly bins | Filled by absolute hour in the same pass. |
| hourly sums to 30d | Bin totals equal the 30 day window totals. |
| unknown token in bins | Counts in the sales bin, no hourly volume. |
| fixed length bins | Events outside are ignored, never written out of range. |
| only via client | No direct HTTP. |

### `listings/listings-counter.service.spec.ts`

| Test | What it proves |
|---|---|
| TC5.1 one pass | Events at 1 h, 3 d, 20 d give 24h=1, 7d=2, 30d=3. |
| TC5.2 inclusive boundary | Exactly 24 h counts. |
| TC5.3 three pages | Three requests, all counted. |
| request params | `event_type=listing`, 50 per page, `after` now minus 30 days on every page. |
| TC5.4 cap | Stops at 400 pages and sets `truncated`. |
| per window, 30d floor | Busy collection exact for 24h and 7d. |
| per window, 7d floor | Cap hit before 7d: 7d and 30d are floors. |
| per window, all floors | Oldest event still inside 24h. |
| per window, no timestamps | Treated as a floor everywhere. |
| reached the end | Not truncated anywhere. |
| TC5.5 exactly 400 pages | No further cursor means not truncated. |
| TC5.6 error on page 7 | Slug keeps earlier counts as stale; others complete. |
| never counted | Reported unavailable, not zero. |
| TC5.7 all fail | Throws, cache not written. |
| TC5.8 overlap | A tick during a run is skipped. |
| guard released | Next tick works after a failed run. |
| TC5.9 normal priority | No priority option. |
| TC5.10 empty | Zeros, not truncated. |
| cache write | `market:listings`, no expiry, with `generatedAt`. |
| removed slug dropped | Not carried forever. |
| no timestamp | Not counted, logged. |
| repeated cursor | Failure, not a loop. |
| empty slug list | Throws. |
| rate limit log | Once at the end. |
| 404 untracked | Marked once, others complete. |
| hourly counts | Filled by absolute hour. |
| hourly sums to 30d | Totals agree. |
| fixed length | Outside events ignored. |
| failed keeps hourly | Earlier hourly counts kept with earlier figures. |
| only via client | No direct HTTP. |

### `stats/stats-aggregator.spec.ts`

| Test | What it proves |
|---|---|
| TC6.1 service sum | Two slugs of one service add up. |
| TC6.2 industry equals services | For every window. |
| TC6.3 zero sales | Average is 0, not NaN. |
| TC6.3b volume with zero sales | Still 0, not Infinity. |
| TC6.4 weighted average | Total over total, not mean of averages. |
| TC6.5 truncation scope | Flags industry and only that collection's service. |
| TC6.5b per window | A 30d only floor flags only the 30d tile. |
| TC6.6 no listings cache | `newListings` 0, rest produced. |
| slug missing from listings | Adds 0 without disturbing others. |
| TC6.7 unknown symbol collection | No volume, sales count. |
| rounding | Two decimals for USD, whole counts. |
| byService | Only services that produced data. |
| TC6.9 coverage | Distinct sorted services and chains. |
| coverage honest | Untracked bsc never claimed. |
| TC6.11 frozen keys | Exact key set for all windows. |
| listingsReady false | Until a listings run produced counts. |
| listingsReady true | Even when every count is a real zero. |
| plain JSON | Survives a round trip. |

### `stats/stats.job.spec.ts`

| Test | What it proves |
|---|---|
| full stats run | Slugs, one price fetch, pull, writes core and payload with no expiry. |
| previous figures passed | Sales counter receives the previous results. |
| TC6.6 no listings | Payload still published with 0, gap logged. |
| cached listings used | Stats run picks up existing listing counts. |
| TC6.8 price failure | Writes nothing, previous payload and `generatedAt` untouched. |
| pull failure | Writes nothing. |
| slug failure | Writes nothing. |
| TC6.12 listings run | Fresh listing counts, volume and sales unchanged. |
| listings failure | Public payload untouched. |
| listings skipped | Recomposes nothing. |
| series written | Built from the same inputs and moment, no expiry. |
| series hourly sales | Carries hourly sales from the cached core. |
| failed run keeps series | Previous series untouched. |
| listings rebuilds series | `listingsReady` turns true without waiting for stats. |
| recompose before core | Writes nothing, returns null. |
| stats overlap | Skipped, guard released after failure. |

### `series/series-builder.spec.ts`

The fixture pins `NOW` to 10:30 UTC on 2 October 2026 and builds bins by absolute hour, with `hourAgo(n)` meaning n hours before the current hour.

| Test | What it proves |
|---|---|
| 24h shape | 24 hourly buckets ending at the end of the current hour, lists equal length. |
| 7d and 30d shape | 7 and 30 daily buckets. |
| first and last bucket | Current hour is the last bucket, 23 hours ago is the first. |
| 24 hours back excluded | The hour just before the window is not in it. |
| daily bucket is 24 hours | Newest day is the last 24 hours, the previous one the 24 before. |
| 30d coverage | 30 daily buckets, oldest first. |
| multiple collections | Summed per bucket for sales, volume and listings. |
| different base hours | Results counted at different moments align by absolute hour. |
| rounding | USD volume to two decimals. |
| only results count | Listings for slugs not in results are ignored. |
| no hourly bins | Old cache entries add nothing. |
| out of range hours | Ignored rather than read past the array. |
| no listings | All zero and `listingsReady` false. |
| listingsReady true | Once counted, even if all zero. |
| listingsReady false | Payload exists but no collection counted. |
| per window truncation | A 30d only floor flags only the 30d series. |
| fallback truncation flag | Old entries use the overall flag. |
| avgSaleUsd per bucket | Volume over sales, null where no sales, never 0, NaN or Infinity. |
| not an average of averages | Day of $1,000 over 1 sale plus $90 over 9 sales averages $109. |
| across collections | Volumes and sales summed before dividing ($300 over 3 sales is $100). |
| avgSaleUsd length | Right length per window, all null with no sales. |
| avgSaleUsd JSON | Nulls survive a round trip. |
| generatedAt | Stamped with the build time. |
| empty results | Full length, all zero. |
| plain JSON | Round trip unchanged. |
| clock moves an hour | Every bucket shifts left by one and `startUnix` by 3600. |

A small oddity: the comment inside the "not an average of averages" test talks about "1 sale of $10 and 9 sales of $90" while the data is 1 sale of $1,000 and 9 sales of $90 (`series-builder.spec.ts:248`); the assertion of 109 is right for the data, the comment is stale.

### `activity/activity-mapper.spec.ts`

| Test | What it proves |
|---|---|
| TC7.12 frozen keys | A sale maps to exactly the contract keys. |
| listing mapping | NFT from `asset`, type from the query, no tx. |
| no order hash as tx | Even if a listing carries a transaction field. |
| empty tx is null | Sales with no hash get null, not "". |
| registry service and chain | Not taken from the event. |
| TC7.2 name fallback | `#` plus first 10 identifier characters. |
| no NFT | "Unnamed", no throw. |
| TC7.3 decimals | 5e17 with 18 decimals is 0.5. |
| 6 decimal token | 500000 USDT units is 0.5. |
| huge quantities | Exact division via ethers. |
| tiny prices | Precision kept, USD not rounded to zero. |
| TC7.4 WETH | Priced as ETH. |
| TC7.5 unknown symbol | Item kept, null USD. |
| TC7.6 no payment | Null price fields, no throw. |
| unusable payment | Null, never NaN. |
| missing symbol | Price kept, no USD. |
| no timestamp | Dropped. |
| malformed event | Never throws. |
| newest first | Sort order. |
| TC7.9 deterministic ties | Same order whatever the input order. |
| reserved sales | Newest N sales kept even when all listings are newer. |
| sorted overall | Sales interleaved by time. |
| few sales | Listings fill the rest. |
| no sales | Just the newest listings. |
| fewer events than slots | All kept. |
| reservation capped | Never more than the feed holds. |
| no mutation | Input array untouched. |

### `activity/activity.job.spec.ts`

| Test | What it proves |
|---|---|
| calls per slug | One sale and one listing call per slug, 20 each, no `after`. |
| TC7.1 70 events | 40 items, descending, 10 newest sales plus the newest of the rest. |
| feed fix | Newest 10 sales survive when every listing is newer. |
| under 40 | Everything kept. |
| tx and USD | Sale has tx, listing none, USD from the run snapshot. |
| one price fetch | Per run, not per event. |
| unknown symbol | Null USD, item kept. |
| TC7.7 one slug fails | One warning line, others feed the result. |
| one call fails | The slug's other call still counts. |
| TC7.8 all fail | Throws, previous feed and `generatedAt` kept. |
| price failure | Writes nothing, no stale price. |
| no collections | Fails, writes nothing. |
| no events, earlier feed | Never blanks an existing feed. |
| no events, no feed | Empty feed written. |
| cache write | `market:activity`, `generatedAt`, no expiry, rate limit logged. |
| no timestamp | Dropped. |
| overlap | Skipped, guard released after failure. |
| 404s untracked | Marked, logged once, not in the failure warning. |
| real failure beside 404 | Warning counts only the real one. |
| all 404 | Run fails, feed not blanked. |
| only via client | No direct HTTP. |

### `market.controller.spec.ts`

These boot a real Nest app with `setGlobalPrefix('api/v1')` and a mocked cache, and call it with supertest.

| Test | What it proves |
|---|---|
| series 200 raw | Cached series served raw, no auth. |
| series shape | Same keys per window, 24, 7 and 30 bucket lists of equal length. |
| series cache only | Polling reads only `market:series`. |
| series 503 | Before the first run, never empty lists. |
| series independent | Stats and activity ready does not make series ready. |
| series public | No guard metadata. |
| activity 200 raw | Raw frozen shape, no auth. |
| TC7.12 activity keys | Every item has exactly the frozen keys. |
| TC7.11 activity cache only | Polling reads only `market:activity`. |
| activity 503 | Before the first feed run. |
| empty feed | A truly empty feed is a 200 with no items. |
| activity public | No guard. |
| independence | Stats ready does not make activity ready. |
| stats 200 raw | No `Response` wrapper. |
| TC6.10 stats cache only | One cache read, nothing else. |
| stats 503 | Clear message, never zeros. |
| stats public | No guard on controller or route. |
| import check | The serving files import no OpenSea, CoinGecko or job code. |

### `market-job-status.service.spec.ts`

| Test | What it proves |
|---|---|
| clean start | No history, all healthy. |
| success | Records start, success, duration, clears earlier error. |
| failure | Message only, counted, never throws. |
| null is skip | Neither success nor failure. |
| skip keeps error | An earlier failure stays visible. |
| TC8.6 unhealthy after three intervals | Not before. |
| never succeeded | Measured from service start. |
| per job thresholds | Activity unhealthy in minutes, listings only after hours. |
| independent jobs | One job's history does not affect another. |

### `market-data.scheduler.spec.ts`

| Test | What it proves |
|---|---|
| own cron per tick | Each method carries its own expression. |
| TC8.3 timings | Stats at :00 :15 :30 :45, listings at its minute, feed every minute. |
| no collision | Listings never starts with stats over a long window. |
| feed cadence | Every 60 s at second 0 for two hours, no gaps. |
| slugs daily | Once a day at 03:00. |
| TC8.2 disabled | No job runs, nothing recorded. |
| enabled | Each tick runs its job and records success. |
| flag per tick | Turning it off stops jobs without restarting timers (in the test the config is mocked; in production `SecretsService` caches forever). |
| unreadable config | Treated as disabled. |
| failure contained | Recorded, logged by message only, never escapes. |
| skip not logged | Only tested with no prior failure. |
| TC8.1 warm up order | Slugs, stats, activity, listings. |
| warm up resilience | One failure does not stop the others. |
| boot timer | Scheduled after the delay, not before. |
| disabled boot | Nothing scheduled. |
| shutdown | Cancels a pending warm up. |
| unref | Timer does not keep the process alive. |
| TC8.10 no key in logs | Even with an error carrying headers. |

### `market-data.health.service.spec.ts`

| Test | What it proves |
|---|---|
| TC8.8 full view | Config summary, job statuses, rate limit snapshot, cache ages. |
| untracked list | With recheck times. |
| TC8.6 stalled job | Overall `healthy` turns false. |
| disabled healthy | Always true when disabled. |
| TC8.10 no secrets | No key, address or stack. |

### `untracked/untracked-collections.service.spec.ts`

| Test | What it proves |
|---|---|
| 404 marks | Returns true so the caller skips its own warning. |
| only 404 | 500, 403, 429, transport errors and plain errors never mark. |
| once per streak | One log per streak, saying it is not a failure. |
| day skip | Skipped for a day, then retried. |
| new streak | A 404 on recheck logs again; a success leaves it unmarked. |
| per collection | Each slug tracked separately. |
| list | Reports marked and recheck times, drops expired entries. |
| no secrets | Only slugs and labels. |

### `market-data.acceptance.spec.ts`

| Test | What it proves |
|---|---|
| TC8.11 single fork | `pm2.config.js` has no multiple instances and no cluster mode. |
| TC8.10 OpenSea key users | Only the config service, its interface and the client mention `openSeaApiKey`. |
| TC8.10 CoinGecko key users | Only the config service, interface and price service. |
| TC8.10 no key in logger calls | No logger line interpolates anything key like. |
| TC8.12 no addresses | No 40 hex character address in any non test file. |
| TC8.9 read path deps | `MarketReadService` depends only on the cache service; the controller only on the read service. |
| TC8.9 thousand reads | 1,000 calls each to stats and activity produce exactly 2,000 cache reads and no network call. |

That last test has two soft spots worth knowing (`market-data.acceptance.spec.ts:71` to `:91`). It spies on the global `fetch`, but every outbound call in this module goes through axios (`httpClient`), which in Node uses the `http` adapter, so the "no network call" assertion would not catch an accidental axios call. And it reads only stats and activity, not the newer `/market/series`, so the "a thousand reads" guarantee has not been extended to the chart route (the controller spec does cover series as a pure cache read).

## Summary of risks specific to the read side and charts

| Where | What happens |
|---|---|
| `series/series-builder.ts:30`, `:36` | Daily buckets are rolling 24 hour blocks ending at the current UTC hour, not calendar days, so day labels are misleading, especially in IST. |
| `series/series-builder.ts:58` to `:63` with `constants/market-data.constants.ts:75` | The newest one or two hourly `newListings` buckets are understated because the listings run is up to an hour old, giving a fake dip at the right edge. |
| `series/series-builder.ts:67` to `:70` | `listingsTruncated` is one flag per window; the oldest 30d listings buckets are understated with no indication of which. |
| `series/series-builder.ts:74` | Average sale price is low when some sales had unpriced tokens, and all USD is at today's price. |
| `stats/stats-aggregator.ts:37` versus `series/series-builder.ts:74` | "No sales" is `0` in the tiles and `null` in the series. |
| `market.controller.ts:14` to `:33` | Raw payloads instead of the app wide `Response` wrapper; unthrottled and without cache headers. |
| `market-data.acceptance.spec.ts:79` | Spies on `fetch` while the code uses axios, and omits the series route. |
| `market-data.health.controller.spec.ts:19` to `:22` | Only asserts `AccessTokenGuard`, so removing `AdminTokenGuard` would not fail a test. |
| `series/series-builder.spec.ts:248` | Test comment contradicts its own data. |
