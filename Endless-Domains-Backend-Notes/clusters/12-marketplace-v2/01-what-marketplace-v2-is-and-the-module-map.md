# 01. What Marketplace V2 Is, and the Module Map

## Why there is a second marketplace at all

If you have already read [the v1 secondary marketplace note](../03-commerce-and-marketplace/03-domain-listing-and-buy-domain-secondary-marketplace.md), you know the shape of the original resale product. A seller sends a `createListing` transaction to the team's own custom smart contract, `NFTDomainsMarketplaceV5`, the frontend then tells the backend "I just did that, here is my `listingId` and transaction hash," the backend writes a `Pending` row into `tbl_domain_listings`, and a reconciliation cron later pulls the transaction receipt off the chain and decodes the contract's own events to decide whether the listing (or a later purchase) really happened. Every listing in v1 is an on chain transaction the seller pays gas for, and every bit of state lives first in a contract the team wrote and maintains themselves.

Marketplace v2, which lives in a brand new folder `src/components/marketplacev2` that did not exist at all at the baseline commit `a131b429` (9 September 2026), throws that model away and adopts the one OpenSea and most modern NFT marketplaces use, Seaport. Seaport is a widely audited, general purpose order settlement contract. Instead of sending a transaction to list a domain, the seller signs a typed data message (an `EIP-712` signature) describing an order: "I offer this one `ERC-721` token, the domain NFT with this `tokenId`, and in exchange I want this many USDT paid to me, plus this many USDT paid to the platform's fee recipient." Signing is free, no gas, no transaction. The signed order is then posted to this backend, which validates it very carefully (the signer really is the verified wallet holding the session, the signature recovers correctly, the price and fee split match the configured fee rate, the seller really owns the NFT on chain right now, the Seaport contract has approval to move it, and so on), stores it in `tbl_marketplacev2_orders`, and serves it to buyers. When a buyer wants the domain, their own wallet submits the signed order straight to the Seaport contract on Polygon, paying in USDT, and Seaport atomically moves the NFT to the buyer and the USDT to the seller and the fee recipient in a single transaction. The backend never sits in the money path at all. It learns that the sale happened afterwards, through a block poller that reads Seaport's `OrderFulfilled`, `OrderCancelled` and `CounterIncremented` events and the domain NFT's `Transfer` and `ApprovalForAll` events off the chain.

So the core difference, in one sentence, is that v1 is "on chain listings in a custom contract, reconciled by cron," while v2 is "off chain signed Seaport orders, validated and stored by the backend, settled on chain by the buyer, and indexed back by a poller." Everything in this cluster follows from that design. The chain is fixed to Polygon mainnet (chain id `137`), the payment token is USDT on Polygon (six decimals, which is why the code talks about "minor units"), the NFT contract is the Unstoppable Domains contract on Polygon (the secret key is literally named `DOMAIN_NFT_ADDRESS_POLYGON_UD`), and the fee is a configurable number of basis points sent to a configurable recipient address.

The old `src/components/marketplace` folder is still there and still imported into `AppModule`, this is a side by side addition, not a replacement. Nothing in v2 imports anything from v1, and the two share no tables. They do share a few general modules (`DomainDetailModule`, `WalletAddressModule`, `AuthModule`, `UserModule`), which is how v2 knows which domains a user owns and which wallets are linked to their account.

## How the work was named: B02, B03, B04, Sprints, and t49

The code comments in this folder read almost like a project diary, and it helps to decode the naming before you open any file. The work was planned as a set of numbered "briefs," each one a feature area, written in the comments as `B-02`, `B-03`, `B-04` and so on, and each brief was then delivered in numbered sprints. A quick count of the comments in `src/components/marketplacev2` and `src/@core` finds brief `B-09` mentioned 83 times, `B-04` 50 times, `B-03` 25 times, `B-05` 11 times, `B-06` three times, and `B-02`, `B-02b` and `B-07` twice each, and sprint mentions from Sprint 1 all the way to Sprint 9. Reading the comments together gives you this mapping.

| Brief | What it is | Where it lives |
|---|---|---|
| B02 (and B02b) | Session wallet verification, the JWT wallet claims and `RequireVerifiedWalletGuard` (B02), then the separate prove wallet flow for email login users (B02b) | `wallet-verification/`, `src/@core/common/guards/require-verified-wallet.guard.ts`, auth and web3 auth changes |
| B03 | Creating and validating Seaport orders, sprint by sprint: Sprint 1 wired in the shared order builder package, Sprint 2 added local checks one through seven, Sprint 3 the three chain checks (eight through ten), Sprint 4 the real `POST /marketplacev2/orders` with persistence | `order/` |
| B04 | The "demo poller," a block poller that reads Seaport and NFT events and applies them to order state | `poller/` |
| B05 | The real public browse API, with search, filters, category tabs and cursor pagination | `order/` (`browse`) |
| B06, B07 | My listings, my purchases, history, and the watchlist | `order/` |
| B09 | OpenSea backed market data (stats, series, activity) | `market-data/` |

The branch names in git use a different prefix, `t49`, which looks like the parent task or epic number. The merged pull requests since the baseline tell the story in order. `feat/t49-b02-jwt-wallet` landed as pull request #944 on 10 September (and the baseline commit itself, `a131b429`, is the merge of #943 from that same branch, so part of B02 was already in flight at baseline, although none of the files this note covers existed yet). `fix/ethers-v6-migration` landed as #946 on 14 September. `feat/t49-b03-sprint1-order-builder-wiring` landed as #947, #948 and #949 between 16 and 17 September. `feat/t49-b04-demo-poller` landed as #951 on 21 September. A long run of `fix/backend-frontend-shared-package-issue` merges (#954 through #962) followed between 25 and 29 September, then `fix/marketplacev2-issues-resovled` (#963), and finally `feature/open-sea-api-integration` (#964 through #968) for B09, up to the current `uat` head `dc1ba3e8` on 2 October. The individual commits are almost all by one developer, Guru, and the very first one, `836d5f89` "Added new marketpalce v2 code" on 10 September, is the commit that created this folder, added `Marketplacev2Module` to `AppModule`, and (more on this below, because it matters) commented out `CronModule`.

The `.env.sample` comment for the chain keys says "Sprint 4," which refers to B03 Sprint 4, the moment real chain reads and persistence arrived and the chain config therefore became mandatory.

## The root module and how it composes everything

```ts
// src/components/marketplacev2/marketplacev2.module.ts
/**
 * Root module for marketplace v2. Feature areas live in their own subfolder
 * (order/ today; wallet-verification/ is B-02's session wallet piece;
 * poller/ is B-04's demo poller; market-data/ is B-09's OpenSea market data)
 * and are wired in as their own Nest module below.
 */
@Module({
    imports: [
        OrderModule,
        WalletVerificationModule,
        PollerModule,
        MarketDataModule
    ],
    exports: [
        OrderModule,
        WalletVerificationModule,
        PollerModule,
        MarketDataModule
    ]
})
export class Marketplacev2Module { }
```

`Marketplacev2Module` has no providers or controllers of its own. It is a pure "barrel" module whose only job is to pull the four feature modules into the application in one line, and it is registered in `src/app.module.ts` as the very last entry of the `imports` array (line 218, imported at the top of the file as `import { Marketplacev2Module } from '@components/marketplacev2/marketplacev2.module'`). The `exports` array exports all four again, which in practice does nothing today because no other module imports `Marketplacev2Module`, but it means any future module that does would get access to whatever those four modules export.

Underneath those four sit three shared, smaller modules that the feature modules import directly, and the reason they exist is documented right in their comments. `ChainConfigModule` (in `config/`) provides the validated chain configuration, covered in full below. `ListingStatusModule` owns `tbl_marketplacev2_listing_status` and `tbl_marketplacev2_listing_history`, and `TransactionModule` owns `tbl_marketplacev2_transactions`. Both of the latter carry nearly identical comments explaining that they are "independent of both OrderModule and PollerModule, imported by both," because the order service (create and cancel) and the poller's application services (fill and invalidate) both need to write to them, and `OrderModule` and `PollerModule` are siblings that must never import each other, since that would risk a circular import (`OrderModule` already pulls in `DomainDetailModule`). This is a nice, deliberate example of the "extract the shared piece into its own module" pattern NestJS recommends for avoiding circular dependencies, rather than reaching for `forwardRef`.

A smaller detail worth knowing: `OrderModule` registers `PollerCursorEntity` in its own `TypeOrmModule.forFeature([...])`, read only, so that `GET /marketplacev2/orders/stats` can read the poller's `lastIndexedBlock` without importing `PollerModule`. Its comment calls this "lastIndexedBlock without a cross module dependency," and `PollerModule` does the mirror image, registering `OrderEntity` because its application services write order status changes directly.

## The full module tree

```text
AppModule (src/app.module.ts)
 ├── ... every pre existing module, including the v1 MarketplaceModule family
 ├── CronModule                      <- COMMENTED OUT since 836d5f89 (see risks)
 └── Marketplacev2Module             (marketplacev2.module.ts, barrel only)
      ├── OrderModule                (order/order.module.ts)
      │    ├── imports: LoggerModule
      │    │            TypeOrmModule.forFeature([OrderEntity, PollerCursorEntity, WatchlistEntity])
      │    │            ChainConfigModule ──────────────┐
      │    │            DomainDetailModule (pre existing)│
      │    │            ListingStatusModule ─────────┐   │
      │    │            TransactionModule ───────┐   │   │
      │    │            WalletAddressModule      │   │   │
      │    ├── controller: OrderController       │   │   │  /marketplacev2/orders/*
      │    └── providers: 'OrderServiceInterface' -> OrderService
      │                   'WatchlistServiceInterface' -> WatchlistService
      │                   'ListingStatusSyncServiceInterface' -> ListingStatusSyncService
      │                   ListingExpiryScheduler (setInterval, every 5 minutes)
      ├── WalletVerificationModule   (wallet-verification/wallet-verification.module.ts)
      │    ├── imports: AuthModule, UserModule, WalletAddressModule, LoggerModule
      │    ├── controller: WalletVerificationController   /marketplacev2/orders/auth/*
      │    └── provider: 'WalletVerificationServiceInterface' -> WalletVerificationService
      ├── PollerModule               (poller/poller.module.ts)
      │    ├── imports: LoggerModule
      │    │            TypeOrmModule.forFeature([PollerCursorEntity, OrderEntity])
      │    │            ChainConfigModule, ListingStatusModule, TransactionModule
      │    │            WalletAddressModule
      │    ├── controller: PollerController               /marketplacev2/poller/*
      │    └── providers: 'PollerServiceInterface' -> PollerService
      │                   SeaportEventApplicationService
      │                   DomainEventApplicationService
      │                   'ChainEventSource' -> ChainEventSourcePoller (setInterval, 15 s tick)
      └── MarketDataModule           (market-data/market-data.module.ts)
           ├── imports: LoggerModule, SecretsModule
           ├── controllers: MarketDataHealthController  /marketplacev2/market-data/*
           │                MarketController            /market/*  (no marketplacev2 prefix!)
           └── providers: MarketDataConfigService, MarketDataCacheService,
                          OpenSeaRequestQueue (factory, one lane), OpenSeaClient,
                          MarketPriceService, SlugResolverService, SalesCounterService,
                          ListingsCounterService, MarketStatsJob, MarketActivityJob,
                          MarketReadService, MarketJobStatusService (factory),
                          MarketDataHealthService, UntrackedCollectionsService,
                          MarketDataScheduler (@Cron jobs)

Shared leaf modules (imported, never import back):
  ChainConfigModule    (config/)          provides CHAIN_CONFIG
  ListingStatusModule  (listing-status/)  tbl_marketplacev2_listing_status, tbl_marketplacev2_listing_history
  TransactionModule    (transaction/)     tbl_marketplacev2_transactions
```

Because NestJS modules are singletons, `ChainConfigModule` being imported by both `OrderModule` and `PollerModule` still produces exactly one instance, so the factory that loads the chain config from AWS runs once per process, not twice.

## Every route prefix, and every route under it

The global prefix set in `main.ts` is `api/v1`, so every path below is really `/api/v1/...`. Other notes in this cluster go deep into each endpoint, this table is the map.

| Method | Path (after `/api/v1`) | Guards | Notes |
|---|---|---|---|
| GET | `/marketplacev2/orders/auth/wallet-nonce` | `AccessTokenGuard` | B02b, issues a prove wallet nonce (see [02](02-wallet-verification-and-the-verified-wallet-guard.md)) |
| POST | `/marketplacev2/orders/auth/prove-wallet` | `AccessTokenGuard` | B02b, verifies the signature and reissues tokens with wallet claims |
| GET | `/marketplacev2/orders/_health/wallet-verified` | `AccessTokenGuard`, `AdminTokenGuard`, `RequireVerifiedWalletGuard` | B02 Sprint 2 throwaway route proving the guard over HTTP |
| POST | `/marketplacev2/orders/_health/validate-local` | `AccessTokenGuard`, `AdminTokenGuard`, `RequireVerifiedWalletGuard` | B03 Sprint 2, checks one through seven, no persistence |
| POST | `/marketplacev2/orders/_health/validate-chain` | `AccessTokenGuard`, `AdminTokenGuard`, `RequireVerifiedWalletGuard` | B03 Sprint 3, checks one through ten, no persistence |
| GET | `/marketplacev2/orders/_health` | `AccessTokenGuard`, `AdminTokenGuard` | diagnostic |
| GET | `/marketplacev2/orders/_health/entity` | `AccessTokenGuard`, `AdminTokenGuard` | diagnostic |
| GET | `/marketplacev2/orders/_health/schema` | `AccessTokenGuard`, `AdminTokenGuard` | dumps table column and index structure |
| GET | `/marketplacev2/orders/_health/chain-config` | `AccessTokenGuard`, `AdminTokenGuard` | returns chain id, the four addresses and `feeBps`, never the RPC URL |
| GET | `/marketplacev2/orders/_health/active` | `AccessTokenGuard`, `AdminTokenGuard` | newest 50 active orders, for the B04 dev test page |
| GET | `/marketplacev2/orders` | none (public) | B05 browse, `Cache-Control: public, max-age=60` |
| GET | `/marketplacev2/orders/stats` | none | |
| GET | `/marketplacev2/orders/transactions/recent` | none | |
| GET | `/marketplacev2/orders/transactions/mine` | `AccessTokenGuard`, `RequireVerifiedWalletGuard` | |
| GET | `/marketplacev2/orders/transactions/:orderHash` | none | hash constrained by regex |
| POST | `/marketplacev2/orders` | `AccessTokenGuard`, `RequireVerifiedWalletGuard` | create a signed Seaport order |
| GET | `/marketplacev2/orders/my-domains` | `AccessTokenGuard` | |
| GET | `/marketplacev2/orders/my-listings` | `AccessTokenGuard`, `RequireVerifiedWalletGuard` | |
| GET | `/marketplacev2/orders/my-purchases` | `AccessTokenGuard`, `RequireVerifiedWalletGuard` | |
| GET | `/marketplacev2/orders/listing-history` | `AccessTokenGuard`, `RequireVerifiedWalletGuard` | |
| POST | `/marketplacev2/orders/listing-status/sync` | `AccessTokenGuard`, plus the express rate limiter from `main.ts` | |
| POST | `/marketplacev2/orders/watchlist` | `AccessTokenGuard` | idempotent add |
| DELETE | `/marketplacev2/orders/watchlist/:id` | `AccessTokenGuard` | |
| GET | `/marketplacev2/orders/watchlist` | `AccessTokenGuard` | |
| POST | `/marketplacev2/orders/:orderHash/cancel` | `AccessTokenGuard`, `RequireVerifiedWalletGuard` | |
| GET | `/marketplacev2/orders/:orderHash` | none | |
| GET | `/marketplacev2/poller/_health` | `AccessTokenGuard`, `AdminTokenGuard` | diagnostic |
| GET | `/marketplacev2/poller/health` | `AccessTokenGuard` | the "real" poller status view |
| GET | `/marketplacev2/market-data/_health` | `AccessTokenGuard`, `AdminTokenGuard` | diagnostic |
| GET | `/market/stats` | none | B09, returns a raw DTO, not the `Response` wrapper |
| GET | `/market/series` | none | same |
| GET | `/market/activity` | none | same |

Three things in that table are worth slowing down on. First, the wallet verification controller is mounted at `marketplacev2/orders/auth`, which is underneath the same `marketplacev2/orders` prefix as `OrderController`, and `OrderController` has catch all style routes like `GET :orderHash`. What stops `GET /marketplacev2/orders/auth/wallet-nonce` from ever being swallowed by a `:orderHash` route is `ORDER_HASH_ROUTE_PATTERN`:

```ts
// src/components/marketplacev2/order/constants/order-route.constants.ts
/**
 * orderHash is always a 0x-prefixed 32-byte hex value (length: 66 on the
 * column). Constraining every :orderHash route param to this pattern makes
 * a malformed hash 404 at the routing layer instead of reaching a service
 * method and throwing a less specific "no order found", and structurally
 * prevents a literal route segment (e.g. "my-domains") from ever being
 * shadowed by a catch-all :orderHash registered ahead of it, in this
 * controller or any future one on the same path.
 */
export const ORDER_HASH_ROUTE_PATTERN = '0x[0-9a-fA-F]{64}';
```

Every `:orderHash` route is declared as `` `:orderHash(${ORDER_HASH_ROUTE_PATTERN})` ``, an Express path regex, so the parameter only matches `0x` followed by exactly 64 hex characters. A word like `auth`, `watchlist` or `my-domains` can never match it. This is a lovely trick to remember from the frontend router world too: constrain your dynamic segments and route ordering stops being a source of bugs.

Second, `MarketController` uses `@Controller('market')`, not `marketplacev2/market`, so its three routes live at `/api/v1/market/...`, outside the v2 prefix entirely, and they return their DTO directly rather than the house `new Response(message, result)` envelope. A frontend calling them should not expect `result` to wrap the payload.

Third, the "health" routes. Every route whose path starts with `_health` is diagnostic scaffolding, the comment block at the top of `OrderController` says so plainly: "temporary diagnostic scaffolding, not a permanent public API, they exist so QA has something to hit over HTTP (TC1.2, TC2.1/TC2.4, TC3.1/TC3.6, TC4.8) before the real `B-03` endpoints land." They are now locked behind both `AccessTokenGuard` and `AdminTokenGuard`, which is covered next.

## The health routes and the spec that pins their guards

The `_health` routes started life behind `AccessTokenGuard` only. A later commit (`bdd8e80a`, 1 October, "Added new changes in the marketplacev2") added `AdminTokenGuard` on top of every one of them, and added a dedicated spec file at the root of the module to make sure nobody quietly removes it again.

```ts
// src/components/marketplacev2/health-routes.guards.spec.ts
function guardsOf(controller: { prototype: object }, method: string): unknown[] {
    return Reflect.getMetadata(GUARDS_METADATA, (controller.prototype as Record<string, unknown>)[method] as object) ?? [];
}

describe('marketplacev2 _health routes require AdminTokenGuard', () => {
    const orderHealthMethods = ['walletVerifiedHealth', 'validateLocalHealth', 'validateChainHealth', 'health', 'entityHealth', 'schemaHealth', 'chainConfigHealth', 'activeListingsHealth'];

    it.each(orderHealthMethods)('OrderController.%s keeps AccessTokenGuard and adds AdminTokenGuard', (method) => {
        const guards = guardsOf(OrderController, method);
        expect(guards).toContain(AccessTokenGuard);
        expect(guards).toContain(AdminTokenGuard);
    });

    it('every _health route on OrderController is covered by the list above', () => {
        const healthRoutes = Object.getOwnPropertyNames(OrderController.prototype).filter((name) => {
            const path = Reflect.getMetadata('path', (OrderController.prototype as unknown as Record<string, object>)[name]);
            return typeof path === 'string' && path.startsWith('_health');
        });
        expect(healthRoutes.sort()).toEqual([...orderHealthMethods].sort());
    });
    ...
});
```

This is a pattern worth stealing. Instead of spinning up an HTTP server, it reads the metadata the `@UseGuards()` decorator stores on each method (Nest keeps it under the `GUARDS_METADATA` key, using the `reflect-metadata` library), and asserts on it directly. The tests in this file are:

1. `OrderController.%s keeps AccessTokenGuard and adds AdminTokenGuard`, run once for each of the eight method names (`walletVerifiedHealth`, `validateLocalHealth`, `validateChainHealth`, `health`, `entityHealth`, `schemaHealth`, `chainConfigHealth`, `activeListingsHealth`), eight test cases in total.
2. `every _health route on OrderController is covered by the list above`, which enumerates every method on the controller prototype whose route `path` metadata starts with `_health` and asserts the set equals the hardcoded list. This is the clever one: if someone adds a ninth `_health` route and forgets the admin guard, this test fails because the new route is not in the list, forcing them to add it, at which point test 1 checks its guards.
3. `PollerController._health requires AdminTokenGuard; the real /health status route is unchanged`, asserting `health` has exactly `[AccessTokenGuard, AdminTokenGuard]` and `status` has exactly `[AccessTokenGuard]`.
4. `MarketDataHealthController._health requires AdminTokenGuard`, asserting exactly `[AccessTokenGuard, AdminTokenGuard]`.
5. `public order routes are not affected`, asserting `browse` does not contain `AdminTokenGuard`.

Twelve test cases altogether. Notice the order guarantee too: Nest runs guards in the order listed, so `AccessTokenGuard` (a valid JWT) runs first, then `AdminTokenGuard`, then, on the three routes that have it, `RequireVerifiedWalletGuard`. Calling one of these routes from Postman therefore needs both an `Authorization: Bearer <access token>` header and an `admin-token: <ADMIN_TOKEN>` header.

One honest caveat about `AdminTokenGuard` itself, which predates v2 but now protects all of these routes. It compares `headers['admin-token'] == this.ADMIN_TOKEN` with a loose `==`, at `src/@core/common/guards/admin-token.guard.ts:16`. If `ADMIN_TOKEN` were ever missing from the configuration in some environment, `this.ADMIN_TOKEN` would be `undefined`, and a request with no `admin-token` header at all would also read `undefined`, and `undefined == undefined` is `true`, so the guard would wave everyone through. In this module the `AccessTokenGuard` in front still requires a real user JWT, so it would degrade to "any logged in user" rather than "anyone," but `_health/schema` dumping table structure to any logged in user is exactly what the comment above those routes says it wanted to avoid. A strict `===` plus an explicit "is the configured token non empty" check would close it.

## The new rate limiter line in `main.ts`

```ts
// src/main.ts
const apiCallLimiter = rateLimit({
    windowMs: Number(secrets.API_CALL_LIMIT_RESET_TIME) * 60 * 1000,
    max: Number(secrets.API_CALL_LIMIT),
    message: {
        statusCode: HttpStatus.TOO_MANY_REQUESTS,
        message: ['Too many request for API call from this IP, please try again after ' + secrets.API_CALL_LIMIT_RESET_TIME + ' minutes'],
        error: 'Too Many Requests'
    }
});
app.use('/api/v1/auth/forgot-password', apiCallLimiter);
app.use('/api/v1/domain/detail/refresh_domain', apiCallLimiter);
app.use('/api/v1/marketplacev2/orders/listing-status/sync', apiCallLimiter);
```

Line 85 is new. `POST /marketplacev2/orders/listing-status/sync` triggers `ListingStatusSyncService.sync`, which runs the full on chain domain refresh for the caller (it writes `tbl_domain_detail_bc` through `DomainDetailService`, calling out to the blockchain data providers), so it is an expensive endpoint that an impatient user mashing a "sync" button could turn into a lot of third party API calls. Putting it behind the same `express-rate-limit` limiter as `refresh_domain` makes sense, they do essentially the same heavy work.

There is a subtle consequence worth knowing, though. `apiCallLimiter` is one limiter instance, and `express-rate-limit` keeps one in memory store per instance, keyed by client IP. Mounting the same instance on three paths means all three paths share a single counter per IP. A user who hits sync a few times can exhaust their quota for `forgot-password` too, and vice versa, with a 429 whose message does not hint at why. Before this change that was already true for the first two paths, but adding a third, more frequently clicked path makes it much more likely to bite. Three separate `rateLimit({...})` instances would give each path its own budget. Also note `app.use(path, ...)` matches every HTTP method on that path prefix, so it is the `POST` sync that gets counted, and since `app.enableCors(...)` is registered earlier in `bootstrap`, CORS preflight `OPTIONS` requests are answered before they reach the limiter and do not consume quota.

Two smaller `main.ts` changes in the same diff are pure whitespace (a stray two space line replaced by two blank lines before the Swagger block), nothing functional.

## The chain config loader

Every part of v2 that talks to Polygon needs the same handful of values: which RPC endpoint to use, where the Seaport contract lives, where USDT lives, where the domain NFT contract lives, who receives the platform fee, and how big that fee is. Those live in `config/`, six small files that together are a tidy, test covered example of "validate configuration once at boot and fail loudly."

### What the task brief calls six keys, and what the code really reads

The `.env.sample` added in this pull describes the keys like this:

```ts
// .env.sample (shown as a code block because it is a config file)
# Marketplace V2 chain config (Sprint 4) — not read from process.env directly, these six keys
# must be added to the existing AWS_MANAGER secret's JSON (see loadChainConfig in
# src/components/marketplacev2/config/chain-config.loader.ts). Sample values shown here for
# reference only.
# POLYGON_RPC_URL=<POLYGON_RPC_URL>
# SEAPORT_ADDRESS=<SEAPORT_ADDRESS>
# USDT_ADDRESS=<USDT_ADDRESS>
# DOMAIN_NFT_ADDRESS=<DOMAIN_NFT_ADDRESS>
# FEE_RECIPIENT=<FEE_RECIPIENT>
# FEE_BPS=<FEE_BPS, integer 0-10000>
```

That comment is stale in three separate ways, and if you configure a new environment from it the app will not boot. The real key names the loader reads, from `RawChainConfigSecret` in `chain.config.ts`, are `POL_RPC_URL` (renamed from `POLYGON_RPC_URL` on 29 September, according to the doc comment on that field), `SEAPORT_ADDRESS_POLYGON`, `USDT_ADDRESS_POLYGON`, `DOMAIN_NFT_ADDRESS_POLYGON_UD`, `FEE_RECIPIENT` and `FEE_BPS`. Only the last two match the sample. On top of those six required keys there are now two more optional ones, `POLLER_START_BLOCK_POLYGON` and `POLLER_ENABLED`, which the sample does not mention at all, even though the loader's own comment says "add the real key to the secret (see .env.sample)." And the sample says `FEE_BPS` may be `0` to `10000`, while the validator rejects `0` (the minimum is `1`, for a reason explained below). So the real list of secret keys v2 chain config reads from the `AWS_MANAGER` JSON secret is:

| Secret key | Required? | Validator | Becomes `ChainConfig` field |
|---|---|---|---|
| `POL_RPC_URL` | yes | `validateRpcUrl` | `polygonRpcUrl: string` |
| `SEAPORT_ADDRESS_POLYGON` | yes | `validateChecksumAddress` | `seaportAddress: string` (EIP 55 checksummed) |
| `USDT_ADDRESS_POLYGON` | yes | `validateChecksumAddress` | `usdtAddress: string` |
| `DOMAIN_NFT_ADDRESS_POLYGON_UD` | yes | `validateChecksumAddress` | `domainNftAddress: string` |
| `FEE_RECIPIENT` | yes | `validateChecksumAddress` | `feeRecipient: string` |
| `FEE_BPS` | yes | `validateFeeBps` (integer, 1 to 10000) | `feeBps: number` |
| `POLLER_START_BLOCK_POLYGON` | no, falls back with a warning | `validateStartBlock` when present (positive integer) | `pollerStartBlock: number` |
| `POLLER_ENABLED` | no, falls back with a warning | `"true"` or `"false"`, case insensitive, trimmed | `pollerEnabled: boolean` |

The chain id itself is not configurable. It is a constant:

```ts
// src/components/marketplacev2/config/chain-config.constants.ts
export const CHAIN_CONFIG = 'CHAIN_CONFIG';

/** Fixed for the demo deployment (Polygon mainnet) - not part of the AWS secret. */
export const MARKETPLACEV2_CHAIN_ID = 137;
```

`CHAIN_CONFIG` is the string injection token consumers use with `@Inject(CHAIN_CONFIG)`, the same "string token" style this codebase uses for every service interface. Today it is injected by `OrderService`, `ListingExpiryScheduler`, `PollerService`, `ChainEventSourcePoller`, `SeaportEventApplicationService` and `DomainEventApplicationService`. `MARKETPLACEV2_CHAIN_ID` is used to build every `JsonRpcProvider` (`new JsonRpcProvider(this.chainConfig.polygonRpcUrl, MARKETPLACEV2_CHAIN_ID, { staticNetwork: true })`, the `staticNetwork` option is an ethers v6 feature that skips the "which network am I on" RPC call on every request), as the chain id inside the `EIP-712` digest when verifying an order signature, and as a real column on `tbl_marketplacev2_transactions` so a second chain would be a data change, not a schema change.

### The shape of the config

```ts
// src/components/marketplacev2/config/chain.config.ts
/**
 * Resolved, validated chain config for the demo deployment. Sourced from the
 * repo's existing shared AWS Secrets Manager secret (AWS_MANAGER) at boot -
 * see chain-config.loader.ts. All six fields are guaranteed present: if any
 * were missing or invalid, loadChainConfig() would have thrown instead of
 * resolving.
 */
export interface ChainConfig {
    polygonRpcUrl: string;
    seaportAddress: string;
    usdtAddress: string;
    domainNftAddress: string;
    feeRecipient: string;
    feeBps: number;
    /** B-04 poller cursor start point - the block the contracts went live at, per environment. Never 0. */
    pollerStartBlock: number;
    /** Whether this environment runs the poller. Set per environment in its own secret - only one environment may poll against a given database (B-04, "single instance only"). */
    pollerEnabled: boolean;
}
```

Two interfaces, on purpose. `RawChainConfigSecret` describes the JSON exactly as it sits in Secrets Manager, every field optional and every value a string (Secrets Manager key value pairs are always strings). `ChainConfig` describes the validated result, every field required and correctly typed. Keeping "untrusted input shape" and "validated domain shape" as two separate types is the same idea as having a form's raw values type and its parsed submission type on the frontend. The doc comment still says "All six fields" although the interface has grown to eight, a small stale comment.

### The validators, one by one

```ts
// src/components/marketplacev2/config/chain-config.validators.ts
const MAX_FEE_BPS = 10000;
// A configured 0 guarantees every listing's fee leg is zero, which Seaport
// reverts on at fill time (see order.service.ts's zero-split check) - the
// floor is 1, not 0, so that failure mode can never be configured in.
const MIN_FEE_BPS = 1;

export function validateChecksumAddress(key: string, value: string | undefined): string {
    if (!value) {
        throw new ChainConfigError(`Chain config is missing required key "${key}"`);
    }
    try {
        return ethers.getAddress(value);
    } catch {
        throw new ChainConfigError(`Chain config key "${key}" is not a valid address: "${value}"`);
    }
}

export function validateRpcUrl(key: string, value: string | undefined): string {
    if (!value) {
        throw new ChainConfigError(`Chain config is missing required key "${key}"`);
    }
    try {
        // eslint-disable-next-line no-new
        new URL(value);
    } catch {
        // Never echo the raw value - an RPC URL commonly carries its API key
        // in the path or query string, and this error reaches boot logs.
        throw new ChainConfigError(`Chain config key "${key}" is not a valid URL`);
    }
    return value;
}

export function validateFeeBps(key: string, value: string | undefined): number {
    if (value === undefined || value === null || value === '') {
        throw new ChainConfigError(`Chain config is missing required key "${key}"`);
    }
    const parsed = Number(value);
    if (!Number.isInteger(parsed) || parsed < MIN_FEE_BPS || parsed > MAX_FEE_BPS) {
        throw new ChainConfigError(`Chain config key "${key}" must be an integer between ${MIN_FEE_BPS} and ${MAX_FEE_BPS}, got: "${value}"`);
    }
    return parsed;
}

export function validateStartBlock(key: string, value: string | undefined): number {
    if (value === undefined || value === null || value === '') {
        throw new ChainConfigError(`Chain config is missing required key "${key}"`);
    }
    const parsed = Number(value);
    if (!Number.isInteger(parsed) || parsed <= 0) {
        throw new ChainConfigError(`Chain config key "${key}" must be a positive integer block number, got: "${value}"`);
    }
    return parsed;
}
```

`validateChecksumAddress` runs the value through `ethers.getAddress`, which both validates a 20 byte hex address and returns it in its canonical mixed case `EIP-55` checksummed form. That is important later, because the order service compares addresses with strict `!==` (for example the offerer against `getAddress(walletAddress)`), so everything has to be normalized to one casing first. If a mixed case value has a wrong checksum, `getAddress` throws, which catches typos. Note that this is the ethers v6 spelling, in v5 it was `ethers.utils.getAddress`, and the error message is deliberately kept identical to the pre migration one, which is exactly what the validators spec asserts.

`validateRpcUrl` only checks that `new URL(value)` parses, and it is careful never to echo the value in its error, because paid RPC URLs carry their API key in the path. That is a thoughtful touch. The weakness is that `new URL` accepts almost anything with a scheme, `foo:bar` or `mailto:x` parse fine, so a typo like a missing `https://` with some other prefix would pass validation and only fail at the first RPC call. Checking `url.protocol` is `https:` (or `http:`/`wss:`) would make it airtight.

`validateFeeBps` uses `Number(value)` plus `Number.isInteger`, which rejects `"12.5"`, `"abc"` and `""`, and enforces the range 1 to 10000 (basis points, so 10000 is 100 percent). The comment explains the floor: a zero fee produces a zero amount consideration item, and Seaport reverts on fill when a consideration amount is zero, so allowing `0` in config would quietly create listings nobody can ever buy. One quirk worth knowing: `Number(" 250 ")` is `250` and `Number("0x10")` is `16`, so a hex looking string would pass, harmless but surprising.

`validateStartBlock` is the same shape for the poller's start block, requiring a positive integer, because a zero would make the very first poller run try to scan Polygon from genesis.

### The two error classes

```ts
// src/components/marketplacev2/config/chain-config.errors.ts
/** A chain config value is missing, malformed, or fails its field validator. */
export class ChainConfigError extends Error {
    constructor(message: string) {
        super(message);
        this.name = 'ChainConfigError';
    }
}

/** AWS Secrets Manager itself is unreachable, denies access, or the secret ID doesn't exist. */
export class ChainSecretsError extends Error {
    constructor(message: string) {
        super(message);
        this.name = 'ChainSecretsError';
    }
}
```

These are plain `Error` subclasses, not Nest `HttpException`s, because they are never meant to reach an HTTP response, they are meant to crash boot with a clear name. The split is useful operationally: `ChainSecretsError` means "go look at IAM, the secret name or the network," while `ChainConfigError` means "the secret was fetched fine, but its contents are wrong."

### The loader itself

```ts
// src/components/marketplacev2/config/chain-config.loader.ts
export const TEMPORARY_FALLBACK_POLLER_START_BLOCK = 93_840_000;

function validateRaw(raw: RawChainConfigSecret): ChainConfig {
    return {
        polygonRpcUrl: validateRpcUrl('POL_RPC_URL', raw.POL_RPC_URL),
        seaportAddress: validateChecksumAddress('SEAPORT_ADDRESS_POLYGON', raw.SEAPORT_ADDRESS_POLYGON),
        usdtAddress: validateChecksumAddress('USDT_ADDRESS_POLYGON', raw.USDT_ADDRESS_POLYGON),
        domainNftAddress: validateChecksumAddress('DOMAIN_NFT_ADDRESS_POLYGON_UD', raw.DOMAIN_NFT_ADDRESS_POLYGON_UD),
        feeRecipient: validateChecksumAddress('FEE_RECIPIENT', raw.FEE_RECIPIENT),
        feeBps: validateFeeBps('FEE_BPS', raw.FEE_BPS),
        pollerStartBlock: resolvePollerStartBlock(raw.POLLER_START_BLOCK_POLYGON),
        pollerEnabled: resolvePollerEnabled(raw.POLLER_ENABLED)
    };
}

function resolvePollerEnabled(value: string | undefined): boolean {
    if (value === undefined || value === null || value === '') {
        const fallback = process.env.NODE_ENV?.trim() === 'production';
        console.warn(`[chain-config] POLLER_ENABLED is missing from the AWS secret - falling back to ${fallback} (NODE_ENV=${process.env.NODE_ENV?.trim() || 'unset'}). Add the key to the secret.`);
        return fallback;
    }
    const normalized = String(value).trim().toLowerCase();
    if (normalized === 'true') return true;
    if (normalized === 'false') return false;
    throw new ChainConfigError(`POLLER_ENABLED must be "true" or "false", got "${value}"`);
}

function resolvePollerStartBlock(value: string | undefined): number {
    if (value === undefined || value === null || value === '') {
        console.warn(`[chain-config] POLLER_START_BLOCK_POLYGON is missing from the AWS secret - falling back to ${TEMPORARY_FALLBACK_POLLER_START_BLOCK} temporarily. ...`);
        return TEMPORARY_FALLBACK_POLLER_START_BLOCK;
    }
    return validateStartBlock('POLLER_START_BLOCK_POLYGON', value);
}

export async function loadChainConfig(): Promise<ChainConfig> {
    const secretId = process.env.AWS_MANAGER;
    if (!secretId) {
        throw new ChainConfigError('Missing required environment variable "AWS_MANAGER"');
    }

    const secretsManager = new SecretsManager({ region: 'us-east-1' });
    let secretString: string | undefined;
    try {
        const data = await secretsManager.getSecretValue({ SecretId: secretId });
        secretString = data.SecretString;
    } catch (err) {
        const awsErrorName = err?.name ?? err?.code ?? 'UnknownError';
        throw new ChainSecretsError(`Unable to fetch chain config secret "${secretId}": AWS returned ${awsErrorName} (${err?.message ?? String(err)})`);
    }

    if (!secretString) {
        throw new ChainSecretsError(`Chain config secret "${secretId}" has no SecretString payload`);
    }

    let raw: RawChainConfigSecret;
    try {
        raw = JSON.parse(secretString) as RawChainConfigSecret;
    } catch (err) {
        throw new ChainConfigError(`Chain config secret "${secretId}" is not valid JSON: ${err?.message ?? String(err)}`);
    }

    return validateRaw(raw);
}
```

Walk through it as a sequence. The loader reads `process.env.AWS_MANAGER` (the name of the shared secret every other module already uses) and throws `ChainConfigError` immediately if it is not set, before any AWS call. It creates a plain `SecretsManager` client pinned to `us-east-1`, the same hard pin `SecretsService` uses, and the loader's long doc comment says it deliberately mirrors `src/components/aws-secrete/loadSecrets.ts` and `src/@core/utils/secrets/secrets.service.ts` rather than inventing a new abstraction. Any AWS failure (secret not found, access denied, network) becomes a `ChainSecretsError` carrying the AWS error's `name` (for example `ResourceNotFoundException`). An empty `SecretString` (which would happen if the secret were stored as binary) is also a `ChainSecretsError`. Invalid JSON becomes a `ChainConfigError` naming the secret id. Then `validateRaw` runs every validator in a fixed order and builds the typed object. Because each validator throws on the first problem, you see one error at a time: fix `POL_RPC_URL`, reboot, and only then discover `FEE_BPS` is also wrong. Collecting all problems into one message would be friendlier, but one at a time is simpler and still clear.

The two poller keys deliberately do not throw when absent. `POLLER_START_BLOCK_POLYGON` falls back to `TEMPORARY_FALLBACK_POLLER_START_BLOCK` (93,840,000) with a console warning, and the long comment above that constant is candid that this is "not a real deployment block," only there "so local/stage boot isn't blocked," and that the poller will "spend a long time catching up in MAX_RANGE sized steps" if it is ever really used. A present but malformed value still throws exactly as before. `POLLER_ENABLED` falls back to `true` only when `NODE_ENV` trimmed equals `production` (PM2's `pm2.config.js` sets `NODE_ENV: 'production'` on the servers), and to `false` everywhere else. The comment explains a real incident behind this: "On `2026-09-29` a developer's local backend was polling against the shared UAT database alongside the UAT server, and the two kept overwriting the one cursor row, the cursor jumped backward and the lag never dropped." The `.trim()` exists because the Windows form of `npm run start:dev`, `set NODE_ENV=dev && ...`, leaves a trailing space in the variable (`"dev "`), a classic Windows shell gotcha. Both fallbacks use `console.warn`, not `CustomLoggerService`, because this runs inside a module factory before the logger is necessarily available, which means these warnings appear in PM2's stdout log but not in CloudWatch.

### How it is wired into Nest, and what happens at boot if it is wrong

```ts
// src/components/marketplacev2/config/chain-config.module.ts
/**
 * The factory below runs once during Nest's module graph resolution (before
 * app.listen), so a thrown ChainConfigError/ChainSecretsError fails boot with
 * a named error rather than reaching a contract call as undefined. Resolved
 * config is then cached for the process lifetime as an ordinary singleton
 * provider - no repeated AWS calls once the app is up.
 */
@Module({
    providers: [
        {
            provide: CHAIN_CONFIG,
            useFactory: loadChainConfig
        }
    ],
    exports: [CHAIN_CONFIG]
})
export class ChainConfigModule {}
```

`useFactory` with an `async` function is Nest's "async provider" feature. Nest awaits the promise while building the dependency graph inside `NestFactory.create(AppModule)`, and every provider that injects `CHAIN_CONFIG` waits for it. If the promise rejects, `NestFactory.create` rejects, `bootstrap()` in `main.ts` never reaches `app.listen`, and the process exits with the named error in its output, which PM2 will then try to restart in a loop.

That is a good design for v2 itself, but it is worth being very clear about its blast radius: because `Marketplacev2Module` is imported into the root `AppModule` unconditionally, a missing or malformed chain key does not just disable the marketplace, it stops the entire API from booting, including login, checkout, webhooks and everything else. If you are setting up a fresh environment that does not need v2 yet, you still have to put valid values for the six required keys into its secret. A feature flag around `Marketplacev2Module`, or making the loader return a "disabled" config when the keys are absent, would contain that blast radius.

It is also worth knowing that this is a second, separate fetch of the same `AWS_MANAGER` secret at boot. `main.ts` already calls `secretsService.getSecret(process.env.AWS_MANAGER)` for the rate limiter and Moralis, `RefreshTokenStrategy` fetches it in `onModuleInit`, `MarketDataConfigService` fetches it again for the OpenSea keys, and now `loadChainConfig` does too. Each is cheap, but if the secret were ever rotated while the app is running, none of them would see the change until the next restart.

### The two spec files for config

`chain-config.loader.spec.ts` mocks `@aws-sdk/client-secrets-manager` entirely (with a little trick, it stashes the `getSecretValue` mock on the mocked module as `__mockGetSecretValue` so the test can reach it via `jest.requireMock`), sets `process.env.AWS_MANAGER = 'endless-localdev'` before each test, and uses this valid fixture:

```ts
// src/components/marketplacev2/config/chain-config.loader.spec.ts
const VALID_SECRET = {
    POL_RPC_URL: 'https://polygon-rpc.com',
    SEAPORT_ADDRESS_POLYGON: '0x1234567890123456789012345678901234567890',
    USDT_ADDRESS_POLYGON: '0x0987654321098765432109876543210987654321',
    DOMAIN_NFT_ADDRESS_POLYGON_UD: '0x1111111111111111111111111111111111111111',
    FEE_RECIPIENT: '0x2222222222222222222222222222222222222222',
    FEE_BPS: '250',
    POLLER_START_BLOCK_POLYGON: '60000000',
    POLLER_ENABLED: 'true'
};
```

Its test cases, by ID:

| ID | What it proves |
|---|---|
| TC4.1 | Missing `AWS_MANAGER` throws `ChainConfigError` mentioning `AWS_MANAGER`, and Secrets Manager is never called |
| TC4.2 | A `ResourceNotFoundException` from AWS becomes `ChainSecretsError` whose message names that exception |
| TC4.3 | An `AccessDeniedException` becomes `ChainSecretsError` naming it |
| TC4.4 | A non JSON `SecretString` throws `ChainConfigError` naming the secret id `endless-localdev` |
| TC4.5 | Omitting `FEE_RECIPIENT` produces an error mentioning `FEE_RECIPIENT` |
| TC4.6 | A malformed `SEAPORT_ADDRESS_POLYGON` (`0xbadaddress`) produces an error naming the key |
| TC4.7 (twice) | `FEE_BPS` of `10001` and of `-1` both produce an error naming `FEE_BPS` |
| B4.1 | Missing `POLLER_START_BLOCK_POLYGON` does not throw, returns `TEMPORARY_FALLBACK_POLLER_START_BLOCK`, and warns |
| B4.2 | `POLLER_START_BLOCK_POLYGON` of `0` throws |
| B4.3 | Negative start block throws |
| B4.4 | Non integer `12.5` throws |
| (unnamed) | A valid secret resolves to the exact typed object, Secrets Manager is called once with `{ SecretId: 'endless-localdev' }`, and the client is built with `{ region: 'us-east-1' }` |
| POLLER_ENABLED (five cases) | reads `"true"`; reads `"false"` even under `NODE_ENV=production`; missing key falls back to `true` under production with a warning; missing key falls back to `false` under `NODE_ENV='dev '` with a trailing space; a value of `"yes"` throws `ChainConfigError` naming the key |

That is eighteen test cases. B4.1 carries a lovely comment about why it asserts against the exported constant rather than a copied literal: the test started failing on 28 September when the constant was bumped without the test, and asserting against the real export means the two can never drift again. Two small gaps: TC4.5, TC4.6 and TC4.7 only check the message text, not `toBeInstanceOf(ChainConfigError)`, so a regression that threw a plain `Error` with the same text would pass, and there is no test for the empty `SecretString` branch. A third observation: the valid fixture uses all digit addresses, which have no letters to checksum, so the "valid secret" test cannot catch a checksum regression on its own, that job falls to the validators spec.

`chain-config.validators.spec.ts` was added in `26b1f0e4` (14 September, "Upgraded the ethers version") as an "ethers.getAddress rename regression" suite for the v5 to v6 migration. Its four tests check that an all digit lowercase address comes back unchanged, that `0xd8da6bf26964af9d7eed9e03e53415d37aa96045` (Vitalik's well known address, with an independently computed checksum) comes back as `0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045`, that a malformed address throws `ChainConfigError` with the exact pre migration message `Chain config key "SEAPORT_ADDRESS" is not a valid address: "not-an-address"`, and that `undefined` throws `Chain config is missing required key "SEAPORT_ADDRESS"`. Note it uses the old key name `SEAPORT_ADDRESS` as the label, which is fine because the key is just a label passed in.

## The new npm dependencies

Three lines changed in `package.json` that matter for v2.

`@endlessdomains/order-builder` is a brand new dependency pinned to a specific commit of a git repository rather than a version on npm: `"git+https://github.com/Endless-Domains/ed-shared-package.git#855e344aa2be0ebe6287a57de507a71d3be5566e"`. The lockfile records it as version `0.1.0`, license `UNLICENSED`, depending on `ethers ^6.13.4`. This is the shared Seaport order building library extracted from an earlier "seaport polygon spike," and the point of sharing it is that the frontend builds and signs orders with exactly the same code the backend uses to verify them, so the two can never disagree on how an order hash or digest is computed. The backend imports `ItemType`, `OrderType`, `OrderComponents`, `ZERO_ADDRESS`, `ZERO_BYTES32`, `NO_CONDUIT_KEY`, `computeDigest`, `computeOrderHash`, `computeSplit`, `buildOrderComponents` and `OrderError` from it. A smoke spec, `order/order-builder-wiring.smoke.spec.ts`, proves the package resolves under ts jest and that `computeOrderHash` for a fixed fixture still equals `0xec2b15363a346c75217b5d1948432717fe8ea4bc15a7be8993f466063d64cafe`, so any future bump that changes hashing fails immediately. The run of `fix/backend-frontend-shared-package-issue` merges (#954 to #962) is the team getting this shared package to install and build reliably. One risk: `package.json` says `git+https`, but `package-lock.json` resolved it as `git+ssh://git@github.com/...`, and the package is `UNLICENSED`, which suggests a private repository. A CI runner (`buildspec-backend.yml` just runs `npm install`) or a fresh developer machine without GitHub SSH credentials for that organisation will fail to install it.

`an-array-of-english-words` (`^2.0.0`) is a plain list of roughly 275,000 English words. It powers the "Dictionary" and "Brandable" category tabs on the B05 browse endpoint, in `order/util/domain-category.util.ts`, which lazily `require`s it into a `Set` on first use (so code paths that never classify dictionary words never pay the memory cost) and does exact, case insensitive matching, "per product decision, not a substring check."

`ethers` moved from `^5.7.2` to `^6.13.4` (the lockfile resolves `6.17.0`). This was not just for v2, `71fd2fec` and `26b1f0e4` migrated about twenty files across the codebase (contract deployment, NFT collection, ENS, renewals, reputation GM, recent domains, the v1 marketplace event decoder and `listing-id.util.ts`, which now uses native `BigInt` instead of `BigNumber.from`, and web3 auth). The v6 API flattens everything, `ethers.utils.getAddress` becomes `ethers.getAddress`, `ethers.utils.verifyMessage` becomes `ethers.verifyMessage`, `ethers.providers.JsonRpcProvider` becomes `JsonRpcProvider`, and `BigNumber` is replaced by native `bigint`. A search of `src` for `ethers.utils`, `ethers.providers`, `ethers.BigNumber` and `ethers.constants` now finds them only inside comments, so the migration looks complete in code. The older `web3` (`^1.10.0`) package is still present alongside it. The v2 code needed v6 because the shared order builder is written against v6 and uses `bigint` throughout.

Not new in this pull but useful context: the `package.json` also shows NestJS 10, TypeORM 0.3, Jest 29, `@nestjs/cache-manager` 3 with `cache-manager` 7 (TTL in milliseconds), which is newer than what the repository's own `CLAUDE.md` still claims.

## Six new database tables, and how they get created

Every entity under `marketplacev2` uses the `tbl_marketplacev2_` prefix: `tbl_marketplacev2_orders` (`OrderEntity`), `tbl_marketplacev2_watchlist` (`WatchlistEntity`), `tbl_marketplacev2_poller_cursor` (`PollerCursorEntity`), `tbl_marketplacev2_listing_status` (`ListingStatusEntity`), `tbl_marketplacev2_listing_history` (`ListingHistoryEntity`) and `tbl_marketplacev2_transactions` (`TransactionEntity`). No hand written migration files for these were added under `src/one-time-migration` in this pull, and TypeORM `synchronize` is `false`, so the tables rely on the deploy script's "generate migrations, then run them" step described in [the database note](../../04-database-typeorm-and-repositories.md). The other notes in this cluster cover every column.

## Background work inside v2, and the CronModule surprise

Three things in v2 run on their own timers: `ListingExpiryScheduler` (a `setInterval` every five minutes that sweeps stale listings), `ChainEventSourcePoller` (a `setInterval` tick every 15 seconds, gated by `pollerEnabled`), and `MarketDataScheduler` (four `@Cron` jobs, slugs daily at 03:00, stats every fifteen minutes, listings at minute seven of every hour, activity every minute, plus a ten second boot warm up). The two `setInterval` based ones start in `onModuleInit`, so they run as soon as the module boots.

The surprise is in the same first commit that introduced v2. `836d5f89` changed `src/app.module.ts:177` from `CronModule,` to `// CronModule,`. That commit message says nothing about it. `CronModule` (`src/components/cron/`) owns the platform's legacy scheduled jobs: the unverified users reminder every eight hours, `checkWalletBalance` every hour, the pending order reminder every five minutes, the domain expiry notification every day at 2 AM, `checkMintStatusJob` every thirty seconds, and `ReservedDomainSchedulerService`. On the `uat` branch, every one of those is now silently not running. If this was a temporary local convenience (a developer muting noisy jobs while building v2) it needs reverting before this branch is promoted, because mint status checks and pending order reminders are part of the primary purchase flow. `ScheduleModule.forRoot()` is still registered in `AppModule`, so the v2 `@Cron` jobs themselves are unaffected.

## Risks and gaps found while mapping this module

1. `src/app.module.ts:177` comments out `CronModule` (commit `836d5f89`), so on `uat` the mint status check, pending order reminder, wallet balance check, domain expiry email, unverified user reminder and reserved domain scheduler are all disabled.
2. `.env.sample` (its last block) lists the wrong key names (`POLYGON_RPC_URL`, `SEAPORT_ADDRESS`, `USDT_ADDRESS`, `DOMAIN_NFT_ADDRESS`), omits `POLLER_START_BLOCK_POLYGON` and `POLLER_ENABLED`, and says `FEE_BPS` may be `0`, so an environment configured from it fails boot with `ChainConfigError`.
3. `src/components/marketplacev2/config/chain-config.module.ts:16`, through the unconditional import in `app.module.ts:218`, means any chain config problem takes down the whole API, not only the marketplace.
4. `src/main.ts:85` reuses one `apiCallLimiter` instance, so `listing-status/sync`, `forgot-password` and `refresh_domain` share one per IP budget, and clicking sync can lock a user out of password reset for the window.
5. `src/components/marketplacev2/config/chain-config.validators.ts:27` accepts any URL scheme for the RPC endpoint, so a malformed value can pass boot and only fail at the first chain call.
6. `src/@core/common/guards/admin-token.guard.ts:16` uses loose `==` against a possibly undefined `ADMIN_TOKEN`, which would open every `_health` route to any logged in user in an environment missing that key.
7. `package.json` and `package-lock.json` disagree on the transport for `@endlessdomains/order-builder` (`git+https` versus `git+ssh`), and the private, unlicensed repository needs credentials on every CI and developer machine.
8. `src/components/marketplacev2/config/chain.config.ts:4` and the loader comment still say "six fields/keys" although the config now has eight.
9. `MarketController` (`market-data/market.controller.ts:14`) lives at `/market`, outside the v2 prefix, and returns raw DTOs rather than the `Response` envelope, an inconsistency a frontend has to special case.
10. The poller and market data fallbacks warn through `console.warn` in `chain-config.loader.ts:49` and `:61`, so they never reach the CloudWatch log groups and are easy to miss on a server.

## A frontend developer's takeaway

If you build the v2 UI, the mental model is: listing is a signature, buying is a transaction from the buyer's own wallet against Seaport, and the backend is a validator, an order book and an indexer. Your app will need the chain constants the backend uses (chain id `137`, the Seaport, USDT and domain NFT addresses, and `feeBps`), and the safest place to get them is the same secret driven config, which is exactly what `GET /marketplacev2/orders/_health/chain-config` exposes today, although because that route is now admin only you will want a proper public config endpoint, or a build time constant, for the real app. Before any action that touches a wallet (creating, cancelling, viewing "mine" lists) you also need the session to carry a verified wallet, which is the whole subject of [the next note](02-wallet-verification-and-the-verified-wallet-guard.md).
