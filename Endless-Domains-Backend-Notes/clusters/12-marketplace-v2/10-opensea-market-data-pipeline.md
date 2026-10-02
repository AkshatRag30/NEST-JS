# 10. The OpenSea market data pipeline (`B-09`)

## What this module is, in one breath

Everything in `src/components/marketplacev2/market-data/` exists to put four honest numbers on the landing page (volume in USD, number of sales, number of new listings, average sale price) for the whole Web3 domain industry and for each naming service, plus a live feed of the latest sales and listings and, as of the newest commits, a set of chart series that draw those same numbers over time. The internal ticket name is `B-09` and you will see `B-09` at the start of almost every log line the module writes, which is a lovely habit, because grepping CloudWatch for `B-09` gives you this feature's whole story and nothing else.

The work landed in three commits on the `feature/open-sea-api-integration` branch after the baseline `a131b429`: `bdd8e80a` ("Added new changes in the marketplacev2", which brought the whole module), `e3d0c10c` (which added the chart series, the `market:series` cache key, the `GET /market/series` route and the hourly bins inside the sales and listings counters), and `ebb36cd0` ("new avg sale chart", which added the `avgSaleUsd` list to each series window). The module is wired into `Marketplacev2Module` (`src/components/marketplacev2/marketplacev2.module.ts`), which `app.module.ts` imports, so nothing extra is needed to switch it on beyond a secret flag that we will meet shortly.

## What OpenSea is, and why our stats come from it

OpenSea is the biggest general purpose NFT marketplace. The detail that matters for us is that a Web3 domain is, on chain, just an NFT: an ENS name is a token in the ENS BaseRegistrar contract (or the NameWrapper contract if it has been wrapped), an Unstoppable Domains name is a token in UD's contract on Polygon or Base, a SPACE ID `.bnb` name is a token on BNB chain, and so on. Because these are ordinary ERC 721 style tokens, people list them for sale and buy them on OpenSea exactly like they would trade a piece of digital art, and OpenSea indexes every listing and every sale across Ethereum, Polygon, Base, BNB chain and Arbitrum.

That makes OpenSea the single most convenient place to ask "how much domain trading happened in the last 24 hours, across every chain?" Without it we would have to run our own indexer on five chains and decode every marketplace's sale events ourselves, which is a whole product in its own right. The trade off is that we inherit OpenSea's view of the world, including its rate limits, its quirks (listing events come back with `event_type: "order"`, the NFT lives under a different key on sales and listings, two of our collections answer 404), and its blind spots (sales made on other marketplaces that OpenSea does not index will not show up). The module is honest about this in the `source` string it ships with every stats payload:

```ts
// src/components/marketplacev2/market-data/stats/stats-payload.composer.ts
export const MARKET_STATS_SOURCE = 'OpenSea API, USD via CoinGecko at snapshot time (windowed volumes are converted at the current price)';
```

## The guiding rule: a page view never costs an OpenSea call

Before we look at any file, hold onto the architecture in one sentence, because every design decision follows from it. Background jobs talk to OpenSea and CoinGecko on a timetable and write finished payloads into an in memory cache; the public endpoints only ever read that cache. If you are a frontend developer used to "the page calls the API, the API calls the third party", this is the pattern to learn: the third party is called on the server's schedule, not the visitor's, so a traffic spike on the landing page costs zero OpenSea quota. The acceptance spec even enforces this structurally by checking that `MarketReadService` depends on the cache service and nothing else.

## The collections registry: which contracts we track

The list of collections lives in code, but the contract addresses do not. Each row names a Secret Manager key that holds the address:

```ts
// src/components/marketplacev2/market-data/config/market-collections.registry.ts
export const MARKET_COLLECTIONS_REGISTRY: MarketCollectionRegistryEntry[] = [
    { service: 'ENS', openSeaChain: 'ethereum', contractKey: 'MARKET_ENS_ETH_BASE_REGISTRAR_CONTRACT', label: 'ENS BaseRegistrar (ethereum)' },
    { service: 'ENS', openSeaChain: 'ethereum', contractKey: 'MARKET_ENS_ETH_NAME_WRAPPER_CONTRACT', label: 'ENS NameWrapper (ethereum)' },
    { service: 'Unstoppable Domains', openSeaChain: 'matic', contractKey: 'MARKET_UD_POL_CONTRACT', label: 'Unstoppable Domains (matic)' },
    { service: 'Unstoppable Domains', openSeaChain: 'base', contractKey: 'MARKET_UD_BASE_CONTRACT', label: 'Unstoppable Domains (base)' },
    { service: 'SPACE ID', openSeaChain: 'bsc', contractKey: 'MARKET_SPACEID_BNB_CONTRACT', label: 'SPACE ID .bnb (bsc)' },
    { service: 'SPACE ID', openSeaChain: 'arbitrum', contractKey: 'MARKET_SPACEID_ARB_CONTRACT', label: 'SPACE ID .arb (arbitrum)' },
    { service: 'Freename', openSeaChain: 'matic', contractKey: 'MARKET_FREENAME_POL_CONTRACT', label: 'Freename (matic)' },
    { service: 'Freename', openSeaChain: 'bsc', contractKey: 'MARKET_FREENAME_BNB_CONTRACT', label: 'Freename (bsc)' },
    { service: 'Freename', openSeaChain: 'base', contractKey: 'MARKET_FREENAME_BASE_CONTRACT', label: 'Freename (base)' },
    // The spec's "config flag": present in the secret means included, absent means left out.
    { service: 'Unstoppable Domains', openSeaChain: 'ethereum', contractKey: 'MARKET_UD_ETH_CONTRACT', label: 'Unstoppable Domains (ethereum)', optional: true }
];
```

That is nine required pairs plus one optional one. The `service` string is not decoration: it becomes the key of the `byService` object in the stats response, so the landing page depends on it being exactly `ENS`, `Unstoppable Domains`, `SPACE ID` or `Freename`, and the file's own comment warns about that. The `openSeaChain` is typed as `'ethereum' | 'matic' | 'base' | 'bsc' | 'arbitrum'` in `interface/market-collection.interface.ts`, which is OpenSea's own spelling of the chains (note `matic`, not `polygon`, and `bsc`, not `bnb`). The `optional: true` row is the spec's idea of a feature flag without a flag: if `MARKET_UD_ETH_CONTRACT` is present in the secret the row is included, if it is absent the row is silently dropped with no warning.

Why keep addresses out of source? Partly hygiene, and partly because the acceptance spec `TC8.12` greps every non test file of the module for anything matching `0x[0-9a-fA-F]{40}` and fails the build if it finds one. Adding a brand new chain is one line here plus one secret; changing an existing address is a secret edit plus a restart, because `SecretsService` caches the secret in process for the life of the app.

## `MarketDataConfigService`: turning the secret into a config object

The config service reads the same shared secret everything else in the app reads, `process.env.AWS_MANAGER`, through `SecretsService.getSecret`. Here is the whole resolution:

```ts
// src/components/marketplacev2/market-data/config/market-data-config.service.ts
async getConfig(): Promise<MarketDataConfig> {
    const secrets = await this.secretsService.getSecret(process.env.AWS_MANAGER);

    const enabled = this.readString(secrets, 'MARKET_DATA_ENABLED')?.toLowerCase() === 'true';
    const openSeaApiKey = this.readString(secrets, 'OPENSEA_API_KEY');
    const coingeckoApiKey = this.readString(secrets, 'COINGECKO_API_KEY');

    if (enabled && !openSeaApiKey) {
        throw new Error('B-09 market data is enabled (MARKET_DATA_ENABLED=true) but the OPENSEA_API_KEY key is missing from the secret');
    }

    return {
        enabled,
        openSeaApiKey: enabled ? openSeaApiKey : null,
        coingeckoApiKey,
        collections: this.resolveCollections(secrets)
    };
}
```

So the answer to "where does the OpenSea API key come from?" is: the AWS Secrets Manager secret named by the `AWS_MANAGER` environment variable, under the flat key `OPENSEA_API_KEY`, fetched once and then cached in a `Map` inside `SecretsService` (`src/@core/utils/secrets/secrets.service.ts`, which hardcodes region `us-east-1`). The complete list of secret keys this module reads is `MARKET_DATA_ENABLED`, `OPENSEA_API_KEY`, `COINGECKO_API_KEY` (optional, a CoinGecko demo key), and the ten contract keys from the registry. None of them is in the Joi schema in `app-env-validation.ts`; validation happens here instead.

`readString` treats anything that is not a string, or is blank after trimming, as missing, which is why the spec `TC1.3b` confirms that a key set to `"   "` is treated as absent. `MARKET_DATA_ENABLED` is compared case insensitively to `'true'`, so `"TRUE"` works and anything else, including a missing key, means disabled. When disabled, `openSeaApiKey` is forced to `null` even if a key exists, which means the client physically cannot call OpenSea in a disabled environment.

`resolveCollections` walks the registry, looks up each `contractKey`, and validates the value with ethers v6's `isAddress` (the repo is on `ethers ^6.13.4`, not the v5 the root CLAUDE.md lists). A missing required key logs `skipping "<label>", missing Secret Manager key <KEY>`; a value that is not a valid EVM address (including a mixed case address with a wrong checksum, spec `TC1.4b`) logs `value under <KEY> is not a valid EVM address`. Notice what is in those messages: the key name and the label, never the value. That is deliberate and the spec `TC1.4` asserts it.

`onModuleInit` calls `getConfig()` once at boot and logs `B-09 market data: enabled=true, collections resolved=9/10`. Because `getConfig` throws when enabled without a key, the app refuses to boot in that state, and `market-data.module.spec.ts` (`TC1.3`) proves that app init really fails. The reasoning in the comment is good backend instinct: a misconfiguration that only explodes on the first background job run at 3 a.m. is a misconfiguration nobody sees, so fail loudly at startup instead.

There is a real cost hiding in this design though. `getConfig()` is not memoised. It is called by `OpenSeaClient.get` on every single OpenSea request (`opensea.client.ts:97`), by `MarketPriceService.fetchPrices`, by `SlugResolverService`, by the health service, and by the scheduler's `isEnabled` check on every cron tick. The secret itself is cached so there is no network cost, but `resolveCollections` runs every time and logs its warnings every time (`market-data-config.service.ts:64` and `:69`). If one required contract key is missing in an environment, the listings job, which can make over 400 requests in a run, writes over 400 identical "skipping" warnings, and the activity job adds a dozen more every minute. That is a log flood and a CloudWatch cost, and it buries real warnings. The fix would be to resolve the collections once in `onModuleInit` and reuse the result.

## The OpenSea client and its single request lane

### Why a queue at all

Every OpenSea API key on an account draws from one shared rate limit bucket. If the stats job, the listings job and the activity job all fired requests in parallel, they would trip 429s on each other. So the module funnels every call through exactly one lane, `OpenSeaRequestQueue`, registered as a singleton provider with a factory so Nest does not try to inject its constructor arguments:

```ts
// src/components/marketplacev2/market-data/market-data.module.ts
{ provide: OpenSeaRequestQueue, useFactory: () => new OpenSeaRequestQueue() },
```

### How the queue works

The queue keeps two arrays, `high` and `normal`, and a `draining` flag. `enqueue` pushes a job into the right array and kicks `drain()`. `drain` loops while anything is pending: it first waits for a slot, then picks the next job (high before normal, FIFO within each), runs it to completion, and resolves or rejects that job's promise. Only one task ever runs at a time, so concurrency is exactly one.

```ts
// src/components/marketplacev2/market-data/opensea/opensea-request-queue.ts
async acquireSlot(): Promise<void> {
    for (;;) {
        const readyAt = Math.max(this.lastStartedAt + this.minGapMs, this.pausedUntil);
        const wait = readyAt - this.clock.now();
        if (wait <= 0) {
            break;
        }
        await this.clock.sleep(wait);
        // Loop again: a pause may have been extended while we slept.
    }
    this.lastStartedAt = this.clock.now();
}
```

Two rules come out of `acquireSlot`. Starts are at least `MIN_GAP_MS` (300 ms) apart, which caps the lane at roughly 200 requests a minute even when OpenSea answers instantly. And the whole lane can be frozen with `pauseUntil(epochMs)`, which only ever extends an existing pause, never shortens it (`opensea-request-queue.ts:50`). The loop rechecks after sleeping because a pause can be extended mid sleep. A clever little detail in `drain`: it acquires the slot first and picks the job afterwards, so a high priority job that arrived while the lane was waiting jumps ahead of older normal jobs. Priority never interrupts a running task, it only reorders the waiting ones. The `QueueClock` interface (`now`, `sleep`) is injectable so the specs can assert gaps and pauses with a fake clock and zero real waiting.

### The client methods and exactly what they send

`OpenSeaClient` is documented as "the only door to OpenSea", and spec `TC2.12` enforces that `api.opensea.io` appears in exactly one non test source file (`constants/market-data.constants.ts`). Base URL is `OPENSEA_BASE_URL = 'https://api.opensea.io/api/v2'`. The four methods are:

| Method | HTTP call | Query params | Used by |
|---|---|---|---|
| `resolveContract(chain, address)` | `GET /chain/{chain}/contract/{address}` | none | slug resolver, reads `collection` |
| `getCollectionStats(slug)` | `GET /collections/{slug}/stats` | none | the retired `CollectionStatsService` only |
| `getEvents(slug, params)` | `GET /events/collection/{slug}` | `event_type`, `after` (unix seconds), `limit` (clamped to 1..50), `next` (cursor) | sales counter, listings counter, activity job |
| `getCollection(slug)` | `GET /collections/{slug}` | none | nothing in production today (metadata helper) |

Every path segment goes through `encodeURIComponent`. Every request carries the headers `x-api-key: <OPENSEA_API_KEY>` and `accept: application/json`, a 15 second timeout (`OPENSEA_REQUEST_TIMEOUT_MS`, overriding the shared `httpClient` default of 8 seconds because event pages can be slow), and `validateStatus: () => true` so axios never throws on a status code. That last bit matters: the 429 handling needs to read the response headers, and if axios threw, you would be digging them out of an error object that also carries the request config, API key included.

```ts
// src/components/marketplacev2/market-data/opensea/opensea.client.ts
private async get<T>(path: string, params: Record<string, string | number | undefined> | undefined, options?: OpenSeaRequestOptions): Promise<T> {
    const config = await this.configService.getConfig();
    if (!config.enabled || !config.openSeaApiKey) {
        throw new OpenSeaNotConfiguredError();
    }
    const apiKey = config.openSeaApiKey;
    return this.queue.enqueue(() => this.executeWithRetry<T>(path, params, apiKey), options?.priority ?? 'normal');
}
```

### Retry and rate limit behaviour

Inside `executeWithRetry` the loop runs up to `MAX_RETRY_ATTEMPTS = 5` times. Attempt one uses the slot the queue already granted; attempts two to five call `queue.acquireSlot()` themselves so they obey the same 300 ms gap and any pause. The outcomes are:

1. A transport failure (DNS, timeout, connection reset) becomes `OpenSeaRequestError(path, null, code)`, where `code` is only the axios error code string like `ECONNABORTED`. It is not retried. The original axios error is deliberately thrown away because it carries the headers.
2. A 2xx returns `response.data` untouched. The client does no shape validation; the types in `opensea.types.ts` make every field optional and the jobs read defensively.
3. A 429 reads `Retry-After` as an integer number of seconds, waits that long (or `RATE_LIMIT_FALLBACK_WAIT_MS = 30 s` when the header is missing, negative or an HTTP date), capped at `RATE_LIMIT_MAX_PAUSE_MS = 10 minutes`, and retries the same request. The fifth 429 throws `OpenSeaRateLimitError(path, 5)`.
4. Any other status (404, 500, 403) throws `OpenSeaRequestError(path, status)` immediately, no retry. Callers decide what to do.

After every response the client also reads `x-ratelimit-limit`, `x-ratelimit-remaining` and `x-ratelimit-reset` into a snapshot that the health route shows, and when remaining drops below `RATE_LIMIT_PAUSE_THRESHOLD = 5` it pauses the lane until reset:

```ts
// src/components/marketplacev2/market-data/opensea/opensea.client.ts
if (allowPause && remaining !== null && reset !== null && remaining < RATE_LIMIT_PAUSE_THRESHOLD) {
    const untilMs = Math.min(reset * 1000, this.queue.now() + RATE_LIMIT_MAX_PAUSE_MS);
    this.queue.pauseUntil(untilMs);
    this.logger.warn(`B-09 OpenSea rate limit nearly exhausted (remaining=${remaining}), pausing the request lane until reset`);
}
```

A 429 response never sets this pause (`allowPause` is false for it), because the spec says a 429 is governed by `Retry-After` alone.

Two honest risks live here. First, `reset * 1000` assumes `X-RateLimit-Reset` is an absolute unix timestamp in seconds (`opensea.client.ts:161`), and the specs are written to that assumption (`TC2.3` builds `resetSeconds = START/1000 + 3`). Many APIs send seconds remaining instead. If OpenSea ever sends, say, `60`, then `reset * 1000` is 60,000 ms after 1970, which is in the past, so `pauseUntil` is a silent no op and the "nearly exhausted" protection never actually pauses; the lane keeps firing every 300 ms straight into 429s. It is worth confirming against a live response header once and writing the finding into the code comment. Second, the 429 wait happens inside the running task while it holds the lane (`opensea.client.ts:136`, by design, spec "nothing else is sent while a 429 wait is in progress"). With the 10 minute cap and four waits allowed, one stubborn request can freeze the lane for up to 40 minutes, during which even high priority activity calls cannot run. The activity job's health turns unhealthy after only three minutes, so the health route would at least show it.

### The three error classes

`opensea/opensea.errors.ts` defines `OpenSeaNotConfiguredError` (asked to call while disabled or keyless), `OpenSeaRequestError` (non 2xx non 429, or transport failure, with public `path` and `status` fields) and `OpenSeaRateLimitError` (still 429 after five attempts, with `path` and `attempts`). Their messages read like `OpenSea request failed: GET /events/collection/ens status=500`. They contain only the path, which holds a slug or a public contract address, never the query string or headers. The 404 case is especially important because `UntrackedCollectionsService` recognises `error instanceof OpenSeaRequestError && error.status === 404` as "OpenSea does not track this collection".

## Slug resolution: from contract address to OpenSea collection

OpenSea's stats and events endpoints are keyed by collection slug (for example `ens`), not by contract address, so the first job turns each configured pair into a slug. `SlugResolverService.resolveAll()` does this:

1. Reads the `market:slugs` cache, a map keyed `"<chain>:<lowercased address>"` to `{ slug, resolvedAt }`.
2. For each pair, if the cached entry is younger than `SLUG_CACHE_TTL_MS` (24 hours), reuses it. Otherwise calls `resolveContract` and takes `response.collection`.
3. If a refresh fails but an old entry exists, keeps the old slug with a warning. This is why the cache is written with `CACHE_NO_EXPIRY`: freshness is judged per pair from `resolvedAt`, so a stale copy is always there to fall back on.
4. Dedupes by slug. The two ENS contracts really do share slug `ens`, so they collapse into one `ResolvedSlug` and the first pair in registry order owns service, chain and label. Without this, every ENS sale would be counted twice.
5. Writes the slug map back only if something changed, then filters out any slug `UntrackedCollectionsService` currently marks as untracked.

```ts
// src/components/marketplacev2/market-data/stats/slug-resolver.service.ts
resolveAll(): Promise<ResolvedSlug[]> {
    if (!this.inFlight) {
        this.inFlight = this.doResolveAll().finally(() => {
            this.inFlight = null;
        });
    }
    return this.inFlight;
}
```

That `inFlight` promise is a tiny, useful pattern: two jobs calling `resolveAll` at the same moment share one set of OpenSea calls instead of resolving everything twice. Every job (stats, listings, activity) begins by calling `resolveAll`, so in practice it is called every minute by the activity job and is almost always a pure cache hit.

There is one gap: there is no negative caching. A pair that fails to resolve (for example a contract endpoint returning 404, or `collection` coming back empty) has no cache entry, so the next `resolveAll` tries it again (`slug-resolver.service.ts:62` to `:73`). Since the activity job calls `resolveAll` every minute, a permanently unresolvable pair costs one extra OpenSea call and one warning line every minute, forever. Note also that the "slugs" job reports success even when every pair failed and the list is empty, because `[]` is not `null` to the job status tracker.

## The untracked collections service

Two of the nine contracts (SPACE ID on bsc and Freename on bsc) resolve to a slug through the contract lookup, yet every collection endpoint answers 404: OpenSea knows the contract but has no collection page for it. Treating that as an error every minute would waste calls and flood the log, so `UntrackedCollectionsService` keeps an in memory `Map<slug, { label, service, chain, markedAt }>`:

```ts
// src/components/marketplacev2/market-data/untracked/untracked-collections.service.ts
reportIfNotTracked(error: unknown, target: ResolvedSlug): boolean {
    if (!(error instanceof OpenSeaRequestError) || error.status !== 404) {
        return false;
    }
    const existing = this.entries.get(target.slug);
    const stillInside = existing !== undefined && Date.now() < existing.markedAt + UNTRACKED_RECHECK_MS;
    if (!stillInside) {
        this.entries.set(target.slug, { label: target.label, service: target.service, chain: target.chain, markedAt: Date.now() });
        this.logger.warn(
            `B-09 market data: OpenSea has no collection for "${target.label}" (${target.slug}), treating it as not tracked and skipping it for 24 hours`
        );
    }
    return true;
}
```

Only a 404 counts; a 500, 403, 429 or transport failure returns `false` and stays a normal failure. A marked slug is skipped by every job for `UNTRACKED_RECHECK_MS` (24 hours), then tried again; a fresh 404 starts a new streak and logs once more. `list()` feeds the health route with `markedAt` and `recheckAt` as ISO strings. Because coverage in the stats payload is computed from collections that actually produced data, an untracked collection never appears in `coverage`, so the page never claims a chain it does not have. In memory only, so a restart costs one wasted 404 per untracked collection.

## Pricing: turning token amounts into dollars

### The symbol table

```ts
// src/components/marketplacev2/market-data/pricing/symbol-table.constants.ts
export const COINGECKO_SIMPLE_PRICE_URL = 'https://api.coingecko.com/api/v3/simple/price';

export const SYMBOL_TO_COINGECKO_ID: Readonly<Record<string, string>> = {
    ETH: 'ethereum',
    WETH: 'ethereum',
    BNB: 'binancecoin',
    WBNB: 'binancecoin',
    POL: 'polygon-ecosystem-token',
    MATIC: 'polygon-ecosystem-token',
    WPOL: 'polygon-ecosystem-token',
    WMATIC: 'polygon-ecosystem-token'
};

export const STABLECOIN_SYMBOLS: ReadonlySet<string> = new Set(['USDC', 'USDT', 'DAI']);

export const COINGECKO_IDS: readonly string[] = ['ethereum', 'binancecoin', 'polygon-ecosystem-token'];

export const NATIVE_TOKEN_BY_CHAIN: Readonly<Record<OpenSeaChain, string>> = {
    ethereum: 'ETH',
    base: 'ETH',
    arbitrum: 'ETH',
    matic: 'POL',
    bsc: 'BNB'
};
```

Wrapped tokens price as their base token (WETH is ETH), MATIC and POL are the same CoinGecko id since Polygon's rename, and the three stablecoins are pinned at exactly 1 USD without a fetch. `NATIVE_TOKEN_BY_CHAIN` was the fallback when OpenSea's stats endpoint omitted `volume_symbol`; with that endpoint retired it now only matters to the dormant `CollectionStatsService`.

### `MarketPriceService`

A job calls `fetchPrices()` once at the start of its run, getting a frozen `PriceSnapshot { prices, fetchedAt }`, and passes that same snapshot to `toUsd()` for every amount in the run. The request is a single `GET https://api.coingecko.com/api/v3/simple/price?ids=ethereum,binancecoin,polygon-ecosystem-token&vs_currencies=usd`, sent through the shared `httpClient` with its default 8 second timeout, plus the header `x-cg-demo-api-key` only when `COINGECKO_API_KEY` is configured. Every one of the three prices must be a finite positive number or the whole fetch throws (`market-price.service.ts:66`); there is no partial snapshot and no stale fallback price, so a CoinGecko outage makes the stats and activity jobs fail and the previous cache keeps serving, which is the intended behaviour.

```ts
// src/components/marketplacev2/market-data/pricing/market-price.service.ts
toUsd(snapshot: PriceSnapshot, amount: number | string | null | undefined, symbol: string | null | undefined): number | null {
    const normalized = typeof symbol === 'string' ? symbol.trim().toUpperCase() : '';
    const value = typeof amount === 'string' ? Number(amount) : amount;
    if (typeof value !== 'number' || !Number.isFinite(value)) {
        return null;
    }

    if (STABLECOIN_SYMBOLS.has(normalized)) {
        return value;
    }

    const id = SYMBOL_TO_COINGECKO_ID[normalized];
    const price = id ? snapshot.prices[id] : undefined;
    if (price === undefined) {
        this.reportUnknownSymbol(snapshot, normalized || '(missing symbol)');
        return null;
    }
    return value * price;
}
```

An unknown symbol returns `null` and logs once per snapshot, tracked with a `WeakMap<PriceSnapshot, Set<string>>`, which is a neat trick: when the snapshot is garbage collected after the run, its "already reported" set goes with it, so the next run reports again. The WeakMap means no manual cleanup is ever needed.

Things to keep in mind about precision and meaning. USD is at the current CoinGecko price, not the price at sale time, so a 30 day volume moves with ETH's price even if no new sale happened; the source string says so. Sums are plain JavaScript floats accumulated sale by sale and rounded only at the end with `Math.round(x * 100) / 100`, so totals can be off by a cent in the classic `1.005` way; for a marketing tile that is fine, for anything financial it would not be. Any token outside the table (a bridged `USDC.e`, `APE`, a memecoin) adds a sale but zero dollars, which quietly understates both volume and average price; the only signal is a warning log line. And CoinGecko's free tier has its own rate limit; the activity job alone calls `fetchPrices` every minute, so 1,440 calls a day plus 96 from stats, which is why the optional demo key exists.

## Sales and volume: `SalesCounterService`

Originally sales and volume came from OpenSea's `/collections/{slug}/stats`. The team measured it side by side and found it leaves out sales paid in any token other than the collection's main currency (a 5 WETH sale of about $13.6k was missing from an ENS 24h volume of $0.28) and its sale counts disagreed with the events. So they switched to counting sale events directly, and left `CollectionStatsService` in place, unwired, with its tests, because it is the only code reading floor price. This is "master plan open item 12" in the comments.

`pull(slugs, prices, previous)` loops over slugs sequentially, and for each one pages `GET /events/collection/{slug}?event_type=sale&after=<now minus 30 days>&limit=50&next=<cursor>` until the cursor is empty or `SALES_MAX_PAGES = 400` pages are read. Normal priority, so the live feed stays ahead. In one pass over every event it:

1. Skips events with no numeric `event_timestamp` (counted and logged).
2. Parses the payment with the shared `parsePayment`, which divides the wei style integer string by `10^decimals` exactly using ethers' `formatUnits` before converting the final decimal string to a `Number`. This is the right way to handle amounts that exceed `2^53`.
3. Converts to USD with the run's snapshot. A sale with no usable payment or an unknown token still counts as a sale but adds no volume.
4. Drops the sale into an hourly bin (for the charts, see the next note) and into each window whose start it is at or after: 24h, 7d and 30d. The boundary is inclusive, exactly 24 hours ago counts, one second older does not.

```ts
// src/components/marketplacev2/market-data/sales/sales-counter.service.ts
const baseHour = Math.floor(windowStart['30d'] / HOUR_SECONDS);
const hourly = { baseHour, sales: new Array<number>(HOURLY_BINS).fill(0), volumeUsd: new Array<number>(HOURLY_BINS).fill(0) };
```

A repeated pagination cursor throws (protects against an endless loop). A 404 is handed to the untracked service. Any other error is logged and the slug returns `null`. Back in `pull`, a slug that failed keeps its result from the previous successful run if one exists (`sales-counter.service.ts:62` to `:66`), so a blip never makes a whole service vanish from the page. If zero slugs succeeded fresh the run throws, even if old figures exist, so stale data is never rewritten as new. The return shape is `CollectionStatsResult { slug, service, chain, windows, hourly, floor: { price: null, symbol: null } }`.

The "keep previous" rule has no age limit. If Unstoppable Domains on Base fails every run for three days, its three day old `windows['24h']` keeps being summed into the 24h tile as if it were current, while in the chart its hourly bins (placed by absolute hour) slide off the left edge, so the tile and the chart quietly disagree. Nothing in the health payload says which collection is running on old figures. A sensible improvement would be to drop kept results older than, say, three intervals, and surface the list.

## New listings: `ListingsCounterService`

OpenSea's stats have no listing counts at all, so listings are counted the same way from `event_type=listing` events over `LISTINGS_LOOKBACK_DAYS = 30`, 50 per page, up to `LISTINGS_MAX_PAGES = 400`. Listings are far more numerous than sales (ENS sees about 700 a day), so ENS hits the cap: 400 pages times 50 is 20,000 listings, which does not cover 30 days. This is the job that takes "several minutes", which is why it runs hourly.

Because events come newest first, hitting the cap means only the oldest part of the 30 days is missing. So truncation is judged per window:

```ts
// src/components/marketplacev2/market-data/listings/listings-counter.service.ts
const truncated = more && pagesRead >= LISTINGS_MAX_PAGES;
const truncatedWindows = {} as Record<MarketWindow, boolean>;
for (const window of MARKET_WINDOWS) {
    truncatedWindows[window] = truncated && (oldestSeen === undefined || oldestSeen > windowStart[window]);
}
```

A window is a floor only when the oldest event read is still newer than that window's start. ENS ends up truncated for 30d but exact for 24h and 7d, and the frontend shows "20k+" only on the 30d tile. Exactly 400 pages with no further cursor is not truncated (spec `TC5.5`).

The job has its own `running` flag so an overlapping tick returns `null` (skipped), and it merges results with the previous `market:listings` payload: a slug that fails keeps its last complete counts and is listed in `stale`; a slug never counted is listed in `unavailable` rather than reported as zero; a slug no longer in the run is dropped rather than carried forever. A partial count is never written. If every slug fails the run throws and nothing is written. The payload, `ListingsCachePayload { generatedAt, slugs: Record<slug, SlugListingCounts>, unavailable, stale }`, goes to `market:listings` with no expiry. Each `SlugListingCounts` holds `counts`, `truncated`, `truncatedWindows`, `hourly: { baseHour, counts }`, `pagesRead` and `countedAt`. As with sales, there is no age limit on `stale` entries.

## The live feed: `MarketActivityJob`

Every minute, for every slug, the activity job asks for the newest 20 sales and the newest 20 listings (`ACTIVITY_PER_TYPE_LIMIT = 20`, no `after` filter), with `{ priority: 'high' }` so the hourly listings burst cannot starve it. With eight slugs that is 16 calls; with the two bsc collections marked untracked, 12. It fetches prices once, maps every event with `mapActivityEvent`, and picks the final 40 with `selectFeedItems`.

The mapper encodes everything the team learned from the real API: the NFT is under `nft` on sales and `asset` on listings; the item's `type` comes from which query we made, never from the event's own `event_type` (which is `"order"` for listings); `tx` is the transaction hash for sales and always `null` for listings (a listing has only an order hash, which is never exposed); the name falls back to `#` plus the first 10 characters of the token identifier, then to `Unnamed`; `service` and `chain` come from our registry (the event says `polygon`, our registry says `matic`); `priceUsd` is rounded to four decimals because cheap names are worth cents. An event without a timestamp is dropped because it cannot be placed in a newest first list.

```ts
// src/components/marketplacev2/market-data/activity/activity-mapper.ts
export function selectFeedItems(items: MarketActivityItem[], keep: number, reservedSales: number): MarketActivityItem[] {
    const sorted = [...items].sort(compareActivityItems);
    const reserved = sorted.filter((i) => i.type === 'sale').slice(0, Math.min(reservedSales, keep));
    const taken = new Set(reserved);
    const rest = sorted.filter((i) => !taken.has(i)).slice(0, keep - reserved.length);
    return [...reserved, ...rest].sort(compareActivityItems);
}
```

The reservation (`ACTIVITY_RESERVED_SALES = 10`) exists because a plain "newest 40" was 36 listings and 4 sales in a live check, with the newest sale 12 hours old. Now the newest 10 sales always make it in. `compareActivityItems` sorts by timestamp descending, then sales before listings, then service, then name, so equal timestamps never shuffle between runs, which matters for React list rendering. If every call fails, or prices fail, the run throws and the previous feed survives. A successful run that found zero events never blanks a feed that has items; with no earlier feed an empty one is written because that is the truth. The output goes to `market:activity` with no expiry.

## The stats job and recomposition

`MarketStatsJob` is where the pieces meet, and it is worth understanding why there are two intermediate keys:

```ts
// src/components/marketplacev2/market-data/stats/stats.job.ts
async runStatsJob(): Promise<MarketStatsResponse | null> {
    if (this.statsRunning) {
        this.logger.debug('B-09 stats job: previous run still in progress, skipping this tick');
        return null;
    }
    this.statsRunning = true;
    try {
        const slugs = await this.slugResolver.resolveAll();
        const prices = await this.priceService.fetchPrices();
        const previous = await this.cache.get<StatsCoreCache>(MARKET_CACHE_KEYS.STATS_CORE);
        const results = await this.salesCounter.pull(slugs, prices, previous?.results ?? []);

        const core: StatsCoreCache = { generatedAt: new Date().toISOString(), results };
        await this.cache.set(MARKET_CACHE_KEYS.STATS_CORE, core, CACHE_NO_EXPIRY);
        return await this.recompose();
    } finally {
        this.statsRunning = false;
    }
}

async runListingsJob(): Promise<MarketStatsResponse | null> {
    const slugs = await this.slugResolver.resolveAll();
    const counted = await this.listingsCounter.run(slugs);
    if (!counted) {
        return null;
    }
    return this.recompose();
}
```

Sales run every 15 minutes, listings hourly. Rather than one job recalculating everything, each writes its own ingredient (`market:stats-core` for sales and volume, `market:listings` for listings) and then `recompose()` rebuilds the two public payloads, `market:stats` via `composeStatsPayload` and `market:series` via `buildSeries`, from both ingredients at the same `now`. That means the listings job can freshen listing counts without recounting sales, and the stats and series payloads always agree with each other. The cache rule is strict: a key is overwritten only after every step succeeded, so any throw leaves yesterday's numbers in place.

`aggregateStats` sums per collection results into an `industry` block and one block per service, independently (the industry is not a sum of service blocks, both are sums of the same inputs). Average is total volume over total sales, never an average of averages, and zero sales gives `avgSaleUsd: 0` (in the series it is `null`, see the next note). `composeStatsPayload` adds `generatedAt`, `source`, sorted distinct `coverage.services` and `coverage.chains` from collections that produced data, and `listingsReady`, true once at least one collection has a listing count.

Two inconsistencies to know. When no stats run has ever succeeded (for instance CoinGecko was down at boot), `recompose()` returns `null`, so `runListingsJob` returns `null` even though it just wrote fresh listing counts (`stats.job.ts:80` and `:89` to `:91`). The job status service interprets `null` as "skipped", so the listings job never records a success, and after three hours the health route marks it unhealthy while it is actually working fine. And `runListingsJob` is not protected by `statsRunning`, so a listings recompose and a stats recompose can interleave; each reads both keys and writes both payloads, and in the unlucky ordering a recompose built from the older core writes last. With an in memory store the window is a few microtasks, so this is a theoretical risk rather than a practical one.

## The cache layer: where the data actually lives

The cache is in memory. `app.module.ts:120` registers `CacheModule.register({ isGlobal: true, ttl: 60000 })` from `@nestjs/cache-manager ^3.1.3` over `cache-manager ^7.2.9` with no store option, which means the default Keyv in memory store inside the Node process. There is no Redis and no database table. The same global cache is shared with the admin dashboard's `CacheInterceptor` routes, the localization cache and wallet verification, but every `B-09` key is prefixed `market:` so they never collide.

The wrapper exists to defeat one specific trap: the global default TTL is 60 seconds, so a careless `cache.set(key, value)` would make the stats vanish a minute after each run.

```ts
// src/components/marketplacev2/market-data/market-data.cache.service.ts
async set<T>(key: string, value: T, ttlMs: number): Promise<void> {
    if (typeof ttlMs !== 'number' || !Number.isFinite(ttlMs) || ttlMs < 0) {
        throw new Error(`MarketDataCacheService.set requires an explicit non negative TTL in ms (key: ${key})`);
    }
    const envelope: CacheEnvelope<T> = { value, storedAt: Date.now() };
    await this.cache.set(key, envelope, ttlMs);
}
```

The TTL is mandatory, and `CACHE_NO_EXPIRY = 0` means "never expire" (the constants file notes this was verified against `cache-manager` 7.2.9, where an omitted TTL falls back to the default but an explicit 0 is kept). Every value is wrapped in `{ value, storedAt }` so `getWithAge` can tell the health route how old each payload is without the payload carrying its own timestamp.

| Key | Written by | TTL | Contents |
|---|---|---|---|
| `market:slugs` | `SlugResolverService` | none (per pair freshness 24 h via `resolvedAt`) | `{ "<chain>:<address>": { slug, resolvedAt } }` |
| `market:stats-core` | `MarketStatsJob.runStatsJob` | none | `{ generatedAt, results: CollectionStatsResult[] }` |
| `market:listings` | `ListingsCounterService` | none | `ListingsCachePayload` |
| `market:stats` | `recompose()` | none | public `MarketStatsResponse` |
| `market:series` | `recompose()` | none | public `MarketSeriesResponse` |
| `market:activity` | `MarketActivityJob` | none | public `MarketActivityResponse` |
| `market:job-status:<name>` | nobody | n/a | prefix and `marketJobStatusKey()` are defined but unused, job status lives in a `Map` instead |

Because nothing ever expires, there is no cache stampede: no request ever finds an expired key and triggers a refill, and the public routes never refill anything. The flip side is that staleness is invisible to visitors, which is exactly why the health route reports cache ages. And because the cache lives in the process, a restart or deploy wipes everything: the three public routes return 503 for about the 10 second warm up delay plus the first stats and activity runs, and `listingsReady` is false for the several minutes the first listings run takes. It also means the design is single instance only. The acceptance spec `TC8.11` reads `pm2.config.js` and asserts there is no `instances` greater than one and no `cluster` exec mode; a second instance would run every job twice against the same OpenSea bucket and serve different numbers depending on which instance answered. The same logic applies across environments: if both UAT and production set `MARKET_DATA_ENABLED=true` with keys on the same OpenSea account, they share one rate limit bucket, which is why the interface comment says "only one environment may run the jobs at a time".

## The scheduler

```ts
// src/components/marketplacev2/market-data/constants/market-data.constants.ts
export const MARKET_CRON = {
    SLUGS: '0 0 3 * * *',
    STATS: '0 0,15,30,45 * * * *',
    LISTINGS: '0 7 * * * *',
    ACTIVITY: '0 * * * * *'
} as const;

export const WARMUP_DELAY_MS = 10 * 1000;
export const UNHEALTHY_AFTER_INTERVALS = 3;
```

These are six field cron expressions (second first) on `@nestjs/schedule ^4.1.2`, with no `timeZone` option, so they run in the server's local time. Slugs at 03:00:00 daily, stats at :00, :15, :30 and :45 past every hour, listings at minute 7 of every hour (deliberately not a multiple of 15, so the listings burst never starts together with stats), activity at second 0 of every minute.

`MarketDataScheduler` puts the jobs on that clock. The `@Cron` decorators always register, but every tick goes through `runIfEnabled`, which reads `MARKET_DATA_ENABLED` again first, so flipping the secret off stops all OpenSea and CoinGecko traffic (after the secret cache is refreshed, which in practice means a restart, since `SecretsService` caches forever). An unreadable config is treated as disabled rather than a crash. On `onApplicationBootstrap`, an enabled environment sets a 10 second `setTimeout` for `warmUp()`, which runs slugs, stats, activity and then listings last because it is slow; the timer is `unref()`'d so it never keeps the process alive, and `onModuleDestroy` cancels it.

```ts
// src/components/marketplacev2/market-data/market-data.scheduler.ts
private async runIfEnabled(name: MarketJobName, job: () => Promise<unknown>): Promise<void> {
    if (!(await this.isEnabled())) {
        return;
    }
    const ok = await this.statusService.track(name, job);
    const status = this.statusService.getStatus(name);
    if (!ok && status.lastError && status.lastRunAt) {
        this.logger.error(`B-09 ${name} job failed, previous data keeps serving: ${status.lastError}`);
    }
}
```

There is a small logging bug here. `track` returns `false` both for a failure and for a skipped run, and a skipped run deliberately does not clear `lastError` (spec "a skipped run does not hide an earlier failure"). So if an activity run fails, the next run is slow (say the lane is paused), and the tick after that is skipped as overlapping, this code logs "activity job failed" again with the old message (`market-data.scheduler.ts:109`). The spec "a skipped overlapping run is not logged as an error" only covers the case with no earlier failure. Comparing `failures` before and after `track` would fix it.

## Job status tracking

`MarketJobStatusService` keeps an in memory `Map` for the four jobs. It is provided by factory because its constructor takes an optional clock function for tests. `track(name, job)` increments `runs`, records `lastRunAt`, awaits the job, then classifies the outcome: a `null` or `undefined` result is a skip (`skipped++`, no success time), a value is a success (`lastSuccessAt`, `lastError` cleared), a throw is a failure (`failures++`, `lastError` set to the message only, never a stack). It never throws.

`getStatus` computes `healthy` as "time since last success (or since service start if never succeeded) is at most three of this job's own intervals", using `MARKET_JOB_INTERVAL_MS`: slugs 24 h (so 72 h grace), stats 15 min (45 min), listings 60 min (3 h), activity 60 s (3 min). Measuring a never succeeded job from service start means a job that never works goes unhealthy after the same grace as one that stopped.

## The health route

`GET /api/v1/marketplacev2/market-data/_health`, guarded by `AccessTokenGuard` and `AdminTokenGuard`, returns the standard `Response` wrapper around:

```ts
// src/components/marketplacev2/market-data/market-data.health.service.ts
export interface MarketDataHealth {
    module: string;
    status: string;
    enabled: boolean;
    collections: number;
    healthy: boolean;
    jobs: Record<MarketJobName, MarketJobStatus>;
    rateLimit: OpenSeaRateLimitSnapshot;
    cache: Record<'stats' | 'listings' | 'activity', { ageSeconds: number | null }>;
    untracked: UntrackedCollection[];
}
```

`module` is `'market-data'`, `status` is always `'ok'`, `collections` is the count of resolved registry pairs, `healthy` is the AND of every job's `healthy` when enabled and always `true` when disabled, `rateLimit` is `{ limit, remaining, reset, observedAt }` from the last OpenSea response, and `cache` gives the age in seconds of three of the six keys (`market:series` and `market:stats-core` are not reported, so a stale series would not show here directly, although it is always written together with stats). No key material, no addresses, no stacks.

Be aware that `AdminTokenGuard` (`src/@core/common/guards/admin-token.guard.ts:16`) compares `headers['admin-token'] == this.ADMIN_TOKEN` with loose equality. If `ADMIN_TOKEN` were ever missing from config, `undefined == undefined` is true and any logged in user would pass. That is a pre existing guard issue, not something `B-09` introduced, but this route relies on it. The controller spec only asserts `AccessTokenGuard` is present, not `AdminTokenGuard`.

## The whole data flow

```
                     AWS Secrets Manager (secret named by AWS_MANAGER)
       MARKET_DATA_ENABLED, OPENSEA_API_KEY, COINGECKO_API_KEY, MARKET_*_CONTRACT x10
                                        |
                                        v
                    SecretsService (in process Map, cached forever)
                                        |
                                        v
              MarketDataConfigService.getConfig()  (called on every OpenSea request)
                                        |
   +--------------- MarketDataScheduler (@Cron, server time, single PM2 fork) ---------------+
   |  boot + 10 s warm up: slugs -> stats -> activity -> listings                           |
   |  03:00 daily  slugs      :00/:15/:30/:45 stats      :07 hourly listings      every min activity
   +-----+-------------------------+-------------------------+---------------------------+--+
         |                         |                         |                           |
         v                         v                         v                           v
  SlugResolverService      MarketStatsJob.runStatsJob   MarketStatsJob.runListingsJob  MarketActivityJob
  (market:slugs, 24 h      resolveAll + fetchPrices     resolveAll                     resolveAll + fetchPrices
   per pair, inFlight)     + SalesCounterService.pull   + ListingsCounterService.run   20 sales + 20 listings
         |                 (sale events, 30 d, hourly)  (listing events, 30 d,         per slug, HIGH priority
         |                         |                     400 page cap, hourly)          mapActivityEvent
         |                         v                         v                         selectFeedItems(40, 10)
         |                 market:stats-core         market:listings                          |
         |                         \                       /                                  v
         |                          +--- recompose() -----+                            market:activity
         |                          | composeStatsPayload |
         |                          | buildSeries         |
         |                          v                     v
         |                    market:stats          market:series
         |
         +----------> every OpenSea call: OpenSeaClient -> OpenSeaRequestQueue (1 lane, 300 ms gap,
                      high before normal, pause on remaining < 5, 429 retry x5) -> api.opensea.io/api/v2
                      404 -> UntrackedCollectionsService (skip slug 24 h)
                      MarketPriceService.fetchPrices -> api.coingecko.com simple/price (once per run)

   Browser ---> GET /api/v1/market/stats | /series | /activity ---> MarketReadService ---> cache only
                (public, no guard, raw JSON, 503 until first run)
   Admin   ---> GET /api/v1/marketplacev2/market-data/_health (AccessTokenGuard + AdminTokenGuard)
                ---> config summary + job status Map + rate limit snapshot + cache ages + untracked list
```

## Rate limit budget, roughly

It helps to estimate the steady state. Activity is about 12 to 16 calls a minute. Stats is one page per 50 sales per collection per 15 minutes; ENS has about 600 sales a month, so 12 pages, and the whole run is perhaps 20 calls. Listings is the heavy one: ENS alone is 400 pages, plus a few dozen for the others, so roughly 450 calls once an hour, which at 300 ms minimum gap plus response time is several minutes, the "about 8 minutes" the activity job's comment mentions. Add it up and the average is well under 30 requests a minute, but the listings hour has a burst near the 200 per minute ceiling of the lane. The high priority lane is what keeps the feed fresh during that burst. If OpenSea limits are hit regularly, the constants comment suggests stretching the activity interval to 2 minutes first, as a team decision rather than a code change.

## Everything that can go wrong, collected

| Where | What happens |
|---|---|
| `config/market-data-config.service.ts:64`, `:69` | `getConfig` runs collection resolution again on every OpenSea call and every tick, so one missing contract key logs hundreds of identical warnings per listings run. |
| `opensea/opensea.client.ts:161` | `X-RateLimit-Reset` is assumed to be an absolute unix time; if OpenSea sends seconds remaining, the near exhaustion pause silently never happens. |
| `opensea/opensea.client.ts:136` with `constants/market-data.constants.ts:24` | A 429 wait holds the single lane for up to 10 minutes per attempt, up to about 40 minutes total, starving even high priority feed calls. |
| `stats/slug-resolver.service.ts:62` to `:73` | No negative caching: an unresolvable pair costs one OpenSea call and one warning every minute via the activity job. |
| `market-data.scheduler.ts:63` with `market-job-status.service.ts:63` | The slugs job counts as successful even when every pair failed and zero slugs came back. |
| `sales/sales-counter.service.ts:62` to `:66` | A failing collection's previous figures are reused with no age limit, so a frozen 24h number can persist for days while the chart drifts. |
| `listings/listings-counter.service.ts:94` to `:97` | Same for listings `stale` entries, and the health route does not expose `stale` or `unavailable`. |
| `stats/stats.job.ts:80`, `:89` to `:91` | If no stats run has succeeded yet, a successful listings run is recorded as skipped and goes unhealthy after 3 h. |
| `stats/stats.job.ts:74` to `:80` | Listings and stats recompose are not mutually guarded; an older core could theoretically be written last. |
| `market-data.scheduler.ts:109` | A skipped tick after an earlier failure logs the old error again as a fresh failure. |
| `pricing/market-price.service.ts:96` and `stats/stats-aggregator.ts:18` | Float sums with half up rounding at the end; USD at today's price rather than sale time; unknown tokens add sales but no volume. |
| `pricing/symbol-table.constants.ts:24` | Only USDC, USDT and DAI count as stablecoins; bridged variants like `USDC.E` are excluded from USD. |
| `app.module.ts:120` | In memory cache: every deploy empties it, giving 503s for a few seconds and `listingsReady: false` for minutes, and the design breaks with more than one instance or two enabled environments on one OpenSea account. |
| `src/@core/common/guards/admin-token.guard.ts:16` | Loose `==` comparison would let anyone through `_health` if `ADMIN_TOKEN` were unset. |
| `market-data.health.service.ts:56` to `:60` | Health cache ages omit `market:series` and `market:stats-core`. |
| `constants/market-data.cache-keys.ts:13` | `JOB_STATUS_PREFIX` and `marketJobStatusKey` are dead code. |
| `stats/collection-stats.service.ts` | Retired service kept with tests but not provided in the module; harmless but easy to mistake for live code. |

The next note, [`11-market-endpoints-charts-and-the-frontend.md`](11-market-endpoints-charts-and-the-frontend.md), picks up from the cache and walks through the three public endpoints, exactly how each chart is bucketed (including the new average sale price chart), how a React dashboard should consume them, and every spec in the module.
