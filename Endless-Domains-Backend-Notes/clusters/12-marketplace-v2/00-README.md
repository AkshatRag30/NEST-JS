# Cluster 12: Marketplace V2

## What this cluster actually is

This cluster did not exist when the rest of this folder was first written on 10 September 2026, against commit `a131b429`. Everything in it arrived on the `uat` branch over the following three weeks, in roughly forty five commits and pull requests #944 through #968, ending at merge commit `dc1ba3e8` on 2 October 2026. Together they added close to twenty thousand lines, almost all of them inside one brand new folder, `src/components/marketplacev2`. It is now the single largest and most actively developed area of the backend, and also the most heavily tested one, with fifty of the repository's 148 spec files.

The simplest way to hold it in your head is that marketplace v2 is a second, rebuilt secondary marketplace, running alongside the original one from [cluster 03](../03-commerce-and-marketplace/00-README.md) without replacing it. The v1 marketplace (`src/components/marketplace`, the `DomainListing` and `BuyDomainListing` flow) has the frontend send a transaction to Endless Domains' own custom `NFTDomainsMarketplaceV5` contract first, then has the backend record a pending row and wait for a cron job to confirm it. Marketplace v2 turns that around. A seller signs a Seaport style order in their wallet, which costs no gas and creates nothing on chain, and the backend validates and stores that signed order as the listing itself. When a buyer later fills the order on the public Seaport contract on Polygon, paying in USDT, the backend learns about it from an on chain event poller reading contract logs block by block, not from the frontend telling it. On top of that sits a second, independent pipeline pulling collection statistics, sales, listings and activity from OpenSea's API to power the marketplace's stats tiles and charts.

Five ideas are genuinely new to this codebase here, and each one gets its own notes. The first is wallet proof inside the JWT itself: a short signed challenge proves you control a wallet, the proof is written into your access token as `walletAddress` and `walletVerifiedAt` claims, and a new `RequireVerifiedWalletGuard` protects every route that acts on behalf of a wallet. The second is signed off chain orders, Seaport's model, with an order builder shared with the frontend through a private npm package, `@endlessdomains/order-builder`, so both sides compute exactly the same order hash. The third is a durable, cursor based blockchain log poller, a much more serious design than the old `listener` (which, as [cluster 05](../05-blockchain-infrastructure/09-listener-what-it-really-is.md) explains, was never really a listener at all). The fourth is cursor pagination, used here for the first time instead of the offset pagination (`paginateResponse`) used everywhere else in the app. The fifth is a scheduled, rate limited third party data pipeline (OpenSea plus CoinGecko) with an in memory cache and job health tracking.

The same update also bumped `ethers` from v5 to v6 across the entire backend, which touched every older blockchain file, and commented out `CronModule` in `app.module.ts`, which quietly stopped six legacy scheduled jobs on `uat`. Both are covered here in `12`, and the older notes they affect have been corrected in place, each with an "Update from the October 2026 uat pull" section at the bottom.

## How to read this cluster's notes

01 is the map. It explains why v2 exists, how the sprint briefs (B02, B03, B04 and onward, visible in code comments and in the `feat/t49-*` branch names) line up with the code, every submodule and route prefix, the chain config loaded from AWS Secrets Manager, the new dependencies, and the six new database tables. Read this first, even if you plan to read only one other file.

02 covers wallet verification: the nonce and prove wallet flow, how the proof is carried in the JWT, the new guard and its one hour window decision of 15 September 2026, and the exact sequence a React frontend must follow.

03 is a beginner friendly Seaport primer (offer, consideration, order hash, EIP712 signatures, counters, zones, salts) followed by the `tbl_marketplacev2_orders` table column by column, and how the shared order builder package keeps backend and frontend in agreement.

04 walks through creating a listing step by step: the twelve local validation checks with every reject code, the on chain checks (ownership, approvals) with their retry rules, what gets persisted, and every error the frontend can receive.

05 covers the end of a listing's life: soft cancel, expiry, the five minute `setInterval` expiry sweep, the "servable order" rule that decides whether a signature is ever shown to a buyer, and an important caveat about what "cancelled" really means on chain.

06 covers the read side: browsing with filters and categories (dictionary words, brandables, premium TLDs), the "my domains", "my listings" and "my purchases" views, and a deep explanation of cursor pagination compared with the offset pagination you already know, including how to drive it from an infinite scroll.

07 covers the four supporting tables, watchlist, listing status, listing history and transactions: who writes each one (API or poller), the rate limited listing status sync endpoint, and how history and transaction feeds are read.

08 covers the poller's architecture: why it polls logs instead of using websockets, confirmations and block ranges, the single row cursor table, retry and error classification, the boot time topic assertion, per environment enabling, and an honest analysis of what happens with several processes running at once, or with a chain re org.

09 covers exactly what each decoded event (Seaport fills, cancels and counter bumps, plus NFT transfers and approval changes) does to the database, how it stays idempotent, and how it interacts with the API's own cancel and expiry logic.

10 covers the OpenSea market data pipeline: the collections registry, the single lane rate limited OpenSea client, slug resolution, the stats, sales, listings and activity jobs, CoinGecko pricing, the cache, and job health.

11 covers the public `/market` endpoints, how the series builder buckets each chart (including the new average sale chart), how a React dashboard should consume them, and every one of the market data spec files.

12 covers the ethers v5 to v6 migration across the whole codebase, with a translation table built from the real changes, and the arrival of the test suite: how many spec files exist now, how they are structured, how to run them, and what remains untested.

## A few facts worth carrying into every file below

The project `CLAUDE.md` is now out of date on the stack. On `uat` the backend runs NestJS 10, TypeORM 0.3 and ethers 6, not the NestJS 8, TypeORM 0.2 and ethers 5 it lists. CLAUDE.md also says there are no spec files, which was already wrong at the baseline (86 of them) and is even more wrong now.

The six new tables are `tbl_marketplacev2_orders`, `tbl_marketplacev2_watchlist`, `tbl_marketplacev2_listing_status`, `tbl_marketplacev2_listing_history`, `tbl_marketplacev2_transactions` and `tbl_marketplacev2_poller_cursor`. None of them has a committed migration. They exist only because `run/deploy.sh` deletes `src/migration`, generates a fresh migration from the entity diff, and runs it on every deploy, which is covered in the root [database note](../../04-database-typeorm-and-repositories.md).

Every new configuration value is read from the JSON secret named by `AWS_MANAGER`, not from `process.env`. The names the code actually reads are `POL_RPC_URL`, `SEAPORT_ADDRESS_POLYGON`, `USDT_ADDRESS_POLYGON`, `DOMAIN_NFT_ADDRESS_POLYGON_UD`, `FEE_RECIPIENT`, `FEE_BPS`, `POLLER_START_BLOCK_POLYGON`, `POLLER_ENABLED`, `MARKET_DATA_ENABLED`, `OPENSEA_API_KEY`, `COINGECKO_API_KEY` and ten `MARKET_*_CONTRACT` keys. The block of example names in `.env.sample` is stale and uses different names, so trust 01 and 10 over that file.

Marketplace v2 deliberately splits its routes across two prefixes. Almost everything is under `/api/v1/marketplacev2/...`, but the public market data endpoints sit at `/api/v1/market/stats`, `/series` and `/activity`, and they return raw DTOs rather than the usual `{ success, statusCode, message, result }` envelope. A frontend helper that always unwraps `.result` will break on them.

The one thing worth carrying through this whole cluster is the boundary between what is off chain and what is on chain. A signed order sitting in Postgres, a "cancelled" status in a table, and a cursor saying "we have processed up to block N" are all the backend's beliefs about the chain, not the chain itself. Nearly every risk these notes call out comes down to a moment when those beliefs and the real chain can disagree, and a buyer or seller ends up acting on the wrong one.
