# 02. freename: three modules, one registrar

## The shape of the folder before any code

`freename` is the second outside registrar this company resells domains through, alongside Unstoppable Domains. At thirty five files it is the second largest folder in this cluster, and unlike `ud-integration`, its size comes from genuine internal structure rather than one file growing without bound: the folder is split cleanly into three NestJS modules, `auth`, `domain`, and `search`, each with its own controller or none, its own service or several, and its own entities where it needs to remember state in Postgres. Reading all three in order tells a coherent story: authenticate, then register and mint, then, almost as an afterthought, check availability.

## auth: a real OAuth style token lifecycle, self hosted

Freename's API needs a bearer token like most of the other integrations in this cluster, but unlike any of them, that token has to be actively managed: it expires, it can be refreshed, and the refresh token itself has to be stored somewhere safe between requests. `FreenameAuthService` is built around three operations. `generateAccessToken` calls Freename's login endpoint with a username and password read from config, no bearer token required:

```ts
const { data } = await httpClient.post(
    `${this.FREENAME_API_URL}/api/v1/auth/login`,
    { username: clientName, password: clientSecret },
    { headers: { 'Content-Type': 'application/json' } },
);
this.tokenStore.set(data.access_token, data.expires_in);
await this.freenameRefreshTokenRepo.revokeAllActive(this.ENVIRONMENT);
await this.freenameRefreshTokenRepo.saveRefreshToken({ refreshToken: data.refresh_token, ... });
```

The access token lives only in memory, inside `FreenameTokenStore`, a tiny class holding nothing but a token string and an expiry timestamp with a thirty second safety buffer subtracted off, so a token that is about to expire gets treated as already gone rather than being used for one last request that might fail mid flight. The refresh token is the one that gets persisted, and it gets persisted encrypted: `FreenameRefreshTokenRepository` uses AES 256 GCM, generating a fresh random IV for every encrypt call and storing `iv:authTag:ciphertext` as one colon separated string in a `refreshTokenEncrypted` column. `revokeAllActive` is called on every fresh login specifically so that only one refresh token per environment (`sandbox` or `production`) is ever considered active at a time, which keeps a compromised or stale token from lingering around forever.

`getValidAccessToken`, the method every other Freename call actually depends on, ties all of this together with a deliberately layered fallback:

```ts
async getValidAccessToken(): Promise<string> {
    let token = this.tokenStore.get();
    if (token) { return token; }
    try {
        const refreshed = await this.refreshAccessToken();
        return refreshed.accessToken;
    } catch (err) {
        this.logger.warn(`Refresh failed, falling back to login: ${err.message}`);
    }
    const loggedIn = await this.generateAccessToken();
    return loggedIn.accessToken;
}
```

Memory first, then a refresh using the stored refresh token, then a full login as the last resort. Every other Freename facing service in this folder, in both `domain` and `search`, calls this exact method to build its own headers, so this three step fallback is the thing standing between every Freename API call and a hard authentication failure.

## domain: the actual register, mint, and manage lifecycle

`FreenameApiService` is the low level HTTP client for Freename's own domain endpoints, one method per endpoint (`createRegistrant`, `checkAvailability`, `createZone`, `mintZone`, `checkMintStatus`, record management, chain and registrar lookups), each one building its headers through `getValidAccessToken` and sending through `httpClient`. `getZoneByUuid` is worth reading closely because it is the one place in this file that has to fight an API limitation rather than just wrap it: Freename has no "get zone by id" endpoint, so this method pages through `zones/self` up to twenty pages deep, comparing every zone's `uuid` against the one it is looking for, and gives up with a logged warning if it never finds a match.

`FreenameRegistrationService` is where the actual business process lives, and it is the most heavily wired service in this entire cluster, injecting thirteen other interfaces (wallet lookup, user service, web3 transaction logging, refunds, three separate invoice services, Stripe, Coingate, Cryptomus, coupons) because registering a Freename domain after payment touches nearly every other part of the commerce system. `registerDomainAfterPayment` is the heart of it: look up or create a Freename registrant profile for the buyer's wallet, then for each domain in the order, create a lifecycle row, check availability one more time immediately before registering (in case it sold out between checkout and this moment), create the zone on Freename, and record what came back. `finalizeFreenameMint`, called once Freename's own async minting completes, is the payoff of all that invoice wiring: on a successful mint it updates the order, fires internal analytics events, generates a purchase invoice, logs two web3 transaction rows, sends a completion email, and only then considers the order actually finished; on a failed mint it walks a completely different branch that cancels the order and, depending on which payment provider was used, kicks off either a Coingate or a Cryptomus refund with its own confirmation email.

`FreenameLifecycleService` is the plain TypeORM repository wrapper underneath both of the above, tracking every domain through an enum with nine states, from `REQUESTED` through `REGISTERED`, `MINT_PENDING`, `MINT_COMPLETED`, or `MINT_FAILED`. It is worth noticing that this lifecycle status enum is defined a second time, with a different and much shorter set of values, in `domain/enum/freename-lifecycle-status.enum.ts`. Nothing in the folder imports that second file; the real one used everywhere is the one declared inline inside `freename-domain-lifecycle.entity.ts`. It is dead code, but the kind of dead code worth flagging, because a future edit to "the" lifecycle enum has two files it could land in, and only one of them does anything.

## search: the plain availability check, rebuilt from scratch

`FreenameIntegrationService`, in the `search` sub folder, is the one piece of `freename` that actually matches the small chain folders' shared pattern: given a domain name and a TLD, call Freename's search endpoint and map the result into `registered`, `protected`, `price`, and `availableForFree` fields. What is worth noticing is how independently built it is from the other two modules sitting right next to it. It pulls its own `FREENAME_API_URL` out of config a second time rather than sharing a constant with `FreenameApiService`, and it reimplements a private `checkAvailability` helper that calls the exact same `CHECK_ZONE_AVAILABLITY` endpoint `FreenameApiService.checkAvailability` already wraps, used here specifically as a fallback when the search endpoint returns no matching element for a domain:

```ts
if (!element) {
    logger.log(`No elements for ${fullDomain}, falling back to zone availability check`);
    const zoneAvailable = await this.checkAvailability(fullDomain, headers);
    return { name: fullDomain, registered: !zoneAvailable, ... };
}
```

`getFreeNameDomainSuggestion` fans that same search call out across eight hardcoded speculative TLDs (`metaverse`, `hodl`, `satoshi`, `genesis`, `token`, `sat`, `airdrop`, `rwa`) in parallel with `Promise.all`, pairing each search result with its own zone availability check, and quietly drops any result priced at zero, on the reasoning visible right in the code that a zero price result is not a real offer worth showing a buyer.

## What this means for anyone changing freename later

The three module split is a genuinely sensible design on paper, auth concerns separated from the registration lifecycle separated from a read only availability check, but the search module's independent reimplementation of a call `FreenameApiService` already owns is exactly the seam a bug tends to hide in: fix a header, a URL, or an error handling detail in one copy and the other quietly keeps the old behavior. Anyone asked to touch how Freename availability checks work should check both `domain/services/freename-api.service.ts` and `search/freename-integration.service.ts` before assuming a change in one is enough.
