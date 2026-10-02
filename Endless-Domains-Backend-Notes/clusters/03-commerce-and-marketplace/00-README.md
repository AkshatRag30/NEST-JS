# Cluster 03: Commerce and Marketplace

## What this cluster actually is

This is the money cluster. Everything a frontend developer thinks of as "the checkout API" and "the marketplace API" lives here, and it is almost certainly the single most business critical piece of Endless Domains' backend, because it is the part where a mistake costs somebody real cash rather than just a bad user experience.

It helps to notice upfront that this cluster is really two different businesses stitched together under one folder tree, `src/components/marketplace`, plus a handful of sibling top level folders (`cart`, `invoice`, `coupon`, `refund`, `order-management`, `payment-management`, `cancelled-order-diagnostics`). The two businesses are genuinely different in shape, and almost every file you read here will make more sense once you know which of the two it belongs to.

The first business is primary domain registration. A user searches for a brand new domain name, adds it to a shopping cart (`src/components/cart`), optionally applies a coupon (`src/components/coupon`), and checks out through what this codebase calls a "domain order" (`DomainOrderEntity`, owned by `src/components/domain/domain-order`, a sibling component outside this cluster but referenced constantly from inside it). Payment for that order happens through Stripe, CoinGate, or Cryptomus, handled by top level `src/components/webhooks` and the payment gateway integration folders, both of which belong to other agents' clusters, not this one. What this cluster owns on that side of the business is everything downstream and around that core flow: the cart itself, coupon validation, the sales and purchase invoices generated once payment clears, refund bookkeeping, and two pieces of genuinely advanced, real world payment tooling, a reconciliation cron and a cancelled order diagnostics tool, both of which exist specifically to deal with payments that go wrong.

The second business is the secondary marketplace, the part where a domain someone already owns gets listed for resale and bought by someone else, priced and paid for directly in cryptocurrency on chain rather than through a card or a crypto payment gateway. That flow lives almost entirely inside `src/components/marketplace`, specifically `domain-listing` (listing a domain for sale) and `buy-domain` (buying one), backed by an on chain marketplace smart contract whose events get decoded and verified in `src/components/marketplace/abi`. `transaction-cron` reconciles both businesses' pending blockchain transactions on a schedule. A third, smaller business rides on top of the secondary marketplace, paying to promote a listing so it shows up more prominently, handled by `src/components/marketplace/payment` and `src/components/marketplace/webhook`, which is its own small Cryptomus integration, separate from the top level one.

Everything else inside `src/components/marketplace` is smaller supporting cast, watchlists, premium domain flags, domain appraisal, keyword suggestions, marketplace analytics, purchase history, a contact form, and a cross domain login redirect helper. None of it moves money on its own, but several of it feed into things that do (watchlist counts show up in "recent sales" data, premium and promoted flags are set by analytics tags and paid promotion).

## How to read this cluster's notes

01 covers the cart, the very first place a user's intent to buy becomes a database row.

02 covers coupons, which sit between the cart and checkout and change how much the order actually costs.

03 covers the secondary marketplace's own buy and sell flow, `domain-listing` and `buy-domain`, which is genuinely a separate purchase pipeline from the cart based one, on chain rather than card or gateway based.

04 covers `transaction-cron`, the scheduled job that reconciles pending blockchain transactions for both the primary and secondary flows, real world payment reconciliation.

05 covers `cancelled-order-diagnostics`, a read only root cause analysis tool built specifically to answer "why did this order get cancelled and where did the money go."

06 covers `order-management` and `payment-management`, two admin facing reporting layers over data that mostly lives in other clusters, and is explicit about that boundary.

07 covers all three invoice types, purchase, sales, and refund, and what genuinely distinguishes them.

08 covers `refund` (the refund request and status lifecycle) plus the marketplace's own promotion payment and webhook pair, and draws the boundary against the top level `src/components/webhooks` folder and the payment gateway integration clusters that other agents are covering.

09 covers the smaller marketplace features, watchlist, premium and promoted domains, appraisal, keywords, analytics, history, contact us, and the redirect helper, then closes with a section connecting all of this to what a frontend developer building a checkout or marketplace UI is actually doing on the other side of these APIs.

## The one thing worth carrying into every file below

Look for database transactions, or the lack of them, around anything that changes money or order state. This cluster has exactly one real, multi statement `TypeORM` transaction anywhere in it, inside sales invoice number generation. Everywhere else, a purchase or a payment status change happens as a sequence of separate, un-transacted writes, sometimes protected by a single atomic conditional update that acts like a lightweight lock, sometimes not protected at all. That distinction, and where it genuinely matters, is called out explicitly wherever it comes up.

## Update from the October 2026 uat pull

The secondary marketplace described in note 03 of this cluster now has a successor. The October 2026 pull added an entirely new module, `src/components/marketplacev2`, registered in `src/app.module.ts` as `Marketplacev2Module` (first added in commit `836d5f89`, briefly commented out in `1127bfc8`, and restored in `546e26e9`). Instead of the custom `NFTDomainsMarketplaceV5` contract and the per minute reconciliation cron that notes 03 and 04 here describe, v2 is built on the Seaport protocol with signed off chain orders, an on chain event poller, wallet verification, and an OpenSea market data pipeline. It has its own cluster of notes, starting at [../12-marketplace-v2/01-what-marketplace-v2-is-and-the-module-map.md](../12-marketplace-v2/01-what-marketplace-v2-is-and-the-module-map.md), with the order model in [../12-marketplace-v2/03-seaport-primer-and-the-order-entity.md](../12-marketplace-v2/03-seaport-primer-and-the-order-entity.md), the chain reading side in [../12-marketplace-v2/08-the-onchain-event-poller-architecture.md](../12-marketplace-v2/08-the-onchain-event-poller-architecture.md), and the OpenSea data side in [../12-marketplace-v2/10-opensea-market-data-pipeline.md](../12-marketplace-v2/10-opensea-market-data-pipeline.md).

The v1 code in this cluster has not been removed. `DomainListingModule` and `TransactionCheckCronModule` are still imported in `src/app.module.ts` (lines 178 and 180), and the four `@Cron` jobs in `src/components/marketplace/transaction-cron/cron.service.ts` (lines 123, 332, 538 and 575) still run, so for now both marketplaces live side by side. The only v1 files touched by this pull were the two in `src/components/marketplace/abi/`, migrated to ethers v6 in commit `71fd2fec`, which note 03 now covers in its own update section, including the `Interface.parseLog` null behaviour that would otherwise have broken buy confirmations. The new `src/main.ts` line 85 applies the same `apiCallLimiter` used for forgot password to `/api/v1/marketplacev2/orders/listing-status/sync`, the first v2 route to get the IP rate limiter. The codebase wide ethers story and the new test suite are in [../12-marketplace-v2/12-ethers-v6-migration-and-the-new-test-suite.md](../12-marketplace-v2/12-ethers-v6-migration-and-the-new-test-suite.md).
