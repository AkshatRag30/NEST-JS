# 02. Domain Search and Availability, the Busiest Read Path in the API

## Why this file deserves close attention

Every other endpoint in this module gets hit by a user who already has an account and has already decided to buy or manage something. This one gets hit constantly, by anyone typing into the search box on the homepage before they have signed up for anything at all, on every keystroke of a debounced search box in a typical frontend. `DomainSearchService`, at over fourteen hundred lines, is the single largest file in this entire folder, and it is worth understanding well.

## The shape of a search request

A search comes in through `DomainSearchController` at `POST /api/v1/domain/search/availability` or `POST /api/v1/domain/search/suggestion`, both accepting a `DomainSearchDto` with a bare `domainName` and, for availability, an explicit `tld`. Availability answers one exact question, "is `example.crypto` free right now," while suggestion is what actually powers a typical search box, given just the word `example`, it goes and checks that word against every single blockchain and provider this company supports at once, and returns whichever ones came back available.

## Figuring out who even owns a TLD

Before this service can check availability for a given TLD, it has to know which provider issues it, since `.eth` belongs to ENS, `.crypto` belongs to Unstoppable Domains, `.sol` belongs to Bonfida, and so on. That lookup happens here:

```ts
private async checkDomainProvider(tld: string): Promise<string> {
    const cached = this.tldProviderCache.get(tld);
    if (cached) {
        return cached;
    }
    const responseData = await this.domainProviderAndTldService.getByTld(tld);
    if (responseData == undefined) {
        throw new BadRequestException('Tld does not exists in our system.');
    }
    this.tldProviderCache.set(tld, responseData.domain_provider, this.TLD_CACHE_TTL_MS);
    return responseData.domain_provider;
}
```

This is a direct read against the small `tbl_domain_provider_and_tlds` table (covered in the next note), wrapped in a cache, because this mapping barely ever changes and does not deserve a database round trip on every single keystroke a user makes while searching.

## A hand rolled cache, not the app wide `CacheModule`

The root `app.module.ts` registers a global `CacheModule` specifically so that slow, read heavy endpoints across the app can use Nest's built in `CacheInterceptor` with almost no extra code. This service does not use that mechanism at all. Instead it defines its own small bounded, per entry TTL, least recently used cache class right at the top of the file:

```ts
class LRUTTLCache<K, V> {
    private readonly map = new Map<K, { value: V; expiresAt: number }>();
    private readonly maxSize: number;
    ...
    get(key: K): V | undefined {
        const entry = this.map.get(key);
        if (!entry) return undefined;
        if (Date.now() > entry.expiresAt) {
            this.map.delete(key);
            return undefined;
        }
        this.map.delete(key);
        this.map.set(key, entry);
        return entry.value;
    }
    set(key: K, value: V, ttlMs: number): void {
        if (this.map.has(key)) { this.map.delete(key); }
        while (this.map.size >= this.maxSize) {
            this.map.delete(this.map.keys().next().value);
        }
        this.map.set(key, { value, expiresAt: Date.now() + ttlMs });
    }
}
```

The reason this exists instead of a blanket `@UseInterceptors(CacheInterceptor)` on the controller is visible in how the cache keys are built, `${domainName}:${tld}`, a compound key made from the actual request body, not the route path the built in interceptor keys on by default. Three separate caches get built from this one class, `tldProviderCache` for the TLD to provider mapping, `suggestionCache` keyed by name and TLD for suggestion results, and `availabilityCache` keyed the same way for a single exact availability check, each with its own configurable TTL and its own size cap read from environment variables, defaulting to thirty seconds for availability, sixty seconds for suggestions, and ten minutes for the rarely changing TLD map. A periodic sweep, started in `onModuleInit` and cleared in `onModuleDestroy`, proactively clears out expired entries once a minute so memory does not creep up from long lived, high TTL entries that a plain size based eviction alone would not catch fast enough. This is a genuinely more sophisticated caching strategy than the app wide `CacheInterceptor` offers, built specifically because this endpoint's traffic pattern (unique compound keys, external API calls behind each miss) warranted it.

## A single availability check

```ts
async searchDomain(domainSearchDto: DomainSearchDto): Promise<DomainSearchResultDto> {
    domainSearchDto.domainName = domainSearchDto.domainName?.toLowerCase().replace(/\s+/g, '-');
    domainSearchDto.tld = domainSearchDto.tld?.toLowerCase().replace(/\s+/g, '');
    ...
    const cachedAvail = this.availabilityCache.get(availCacheKey);
    if (cachedAvail) { return cachedAvail; }
    const domainProvider = await this.checkDomainProvider(tld);
    if (domainProvider) {
        const domainSearchResult = await this.getDomainSearchResult(domainProvider, domainName, tld);
        ...
        this.availabilityCache.set(availCacheKey, domainSearchResult, this.AVAILABILITY_CACHE_TTL_MS);
        this.storeDomainSearchLog(domainSearchDto, availability, false).catch(...);
        return domainSearchResult;
    }
}
```

Input is normalized (lowercased, spaces turned into hyphens for the name, stripped entirely for the TLD) before anything else happens, then the cache is checked, then, on a miss, the provider is resolved and `getDomainSearchResult` dispatches to one of eleven private methods based on that provider, `getUDDomainSearchResult`, `getENSDomainSearchResult`, `getBonfidaDomainSearchResult`, `getTezosDomainSearchResult`, `getArbitumDomainSearchResult`, `getBnbDomainSearchResult`, `getAptosDomainSearchResult`, `getTonDomainSearchResult`, `getStarknetDomainSearchResult`, `getBoxDomainSearchResult`, and `getFreenameDomainSearchResult`. Every one of these methods talks to a different chain specific service injected into this class, `EnsWeb3Service.checkDomainAvailability`, `ArbWeb3Service.checkDomainAvailability`, `BonfidaIntegrationService.checkavailability`, and so on, each of which is covered by other agents' notes on the blockchain integration folders, this is exactly the handoff point this note is meant to describe. Every one of these results, whether the domain is available or already registered, gets normalized into the same shape by a small shared helper class:

```ts
// src/components/domain/domain-search/get-register-un-register-domain.ts
public async getRegisteredDomain(domainName, tld, domainStatus, domainStatusLabel, redirectUrl, domainProvider, currency, img, network, expiry, owner): Promise<DomainSearchResultDto> { ... }
public async getUnRegisteredDomain(domainName, tld, price, domainStatus, domainStatusLabel, redirectUrl, domainProvider, currency, img, network): Promise<DomainSearchResultDto> { ... }
```

This is the one place the eleven very different provider integrations converge back into a single consistent response shape the frontend can rely on regardless of which of ten different blockchains actually answered the question.

## Suggestions, fanned out across every provider at once

`getDomainSuggestion` is the far busier of the two endpoints, since it is what a live search box calls on every keystroke. Rather than checking one TLD, it asks all of them in parallel:

```ts
const [
    domainSuggestions, freenameSuggestions, ensResult, arbResult, bnbResult,
    bonfidaResult, tezosResult, aptosResult, tonResult, starknetResult, boxResult
] = await Promise.allSettled([
    this.udIntegrationService.getDomainSuggestions(domainSearchDto.domainName, null),
    this.freenameServiceIntegrartion.getFreeNameDomainSuggestion(domainSearchDto.domainName, domainSearchDto.tld),
    this.getENSDomainSearchResult(domainSearchDto.domainName, 'suggestion'),
    ...
]);
```

`Promise.allSettled` rather than `Promise.all` is a deliberate choice, if one provider's API is slow or down, the other nine results should still come back to the user rather than the whole search failing because of one bad dependency, exactly the kind of resilience choice worth noticing in a system that depends on this many third party services at once. Each settled result is then filtered down to only the ones where `domainAvailability.domainStatus === 'AVAILABLE'`, mapped into a shared `DomainSuggestionDto`, and split into exact matches (the searched word plus that provider's TLD, `example.eth`) versus similar matches, with exact matches surfaced first in the response.

## Live pricing on top of static pricing, for ENS specifically

Most providers here return a flat, static price for a domain, defined in a small standalone `GetPriceOfDomainAccordintToDomainProvider` class. ENS is the one exception, because ENS domains are actually auctioned in a way where genuinely short or desirable names can command a real premium set by an on chain price oracle, not a fixed catalog price:

```ts
private async getDomainPriceWithPremium(domainName: string, domainProvider: DomainProvider): Promise<number> {
    const staticPricing = new GetPriceOfDomainAccordintToDomainProvider();
    const staticPrice = staticPricing.getDomainPrice(domainName, domainProvider);
    if (domainProvider !== DomainProvider.ENS) { return staticPrice; }
    try {
        const ONE_YEAR_SECONDS = 31536000;
        const PREMIUM_THRESHOLD = 1.25;
        const livePrice = await this.ensIntegrationService.getRentPriceUsd(domainName, ONE_YEAR_SECONDS);
        return livePrice > staticPrice * PREMIUM_THRESHOLD ? livePrice : staticPrice;
    } catch (err) {
        return staticPrice;
    }
}
```

It only bothers calling out to the live oracle for ENS, and even then only uses that live number if it comes back at least twenty five percent above the static baseline, otherwise it falls back to the simple static price, and if the oracle call itself fails for any reason, it falls back to the static price as well rather than letting a pricing oracle outage break the search endpoint.

## Every search gets logged, quietly and without blocking

Notice the pattern repeated across almost every method in this file:

```ts
this.storeDomainSearchLog(domainSearchDto, availability, false).catch(err =>
    this.customLoggerService.error(`${DomainSearchService.name} - storeDomainSearchLog failed: ${err?.message}`)
);
```

This call is deliberately not awaited, it fires, and whatever happens to it happens in the background, with a `.catch` purely so a logging failure cannot surface as an unhandled promise rejection. Every single search, successful or not, writes a row into `tbl_domain_search_log` (covered in the last note in this cluster), which is what eventually powers the analytics endpoints under `domain/search/analytics`, `searched_names`, and `searched_tld`, letting the business see, across every visitor and every anonymous search, which names and which TLDs people are actually typing.
