# 04. Domain Core Product

## What lives here

`src/components/domain` is the single largest and most important folder in this whole codebase, somewhere around one hundred and forty files, and it earns that size because it is the actual product. Every other folder in this company, the marketplace, the affiliate program, the invoices, the admin panel, exists to support what happens inside this one folder, a person searching for a domain name, buying it, and later managing or renewing it. If you only had time to read one cluster of these notes before your first day on this codebase, this would be the one.

The folder is organized as seven sub modules, each wired into one top level `DomainModule`:

```ts
// src/components/domain/domain.module.ts
@Module({
    imports: [DomainSearchModule, DomainSearchLogModule, DomainDetailModule, DomainOrderModule, DomainProviderAndTldsModule, DomainFavoriteModule, DomainTenureModule]
})
export class DomainModule {}
```

`domain-search` answers the question "is this name available and how much does it cost." `domain-search-log` quietly records every search for analytics. `domain-provider-and-tlds` is the small reference table that tells the rest of the system which company (Unstoppable Domains, ENS, and so on) actually owns a given top level domain like `.crypto` or `.eth`. `domain-order` is the purchase and checkout flow. `domain-detail` is what a user actually owns, its list view, and the endpoint that goes and refreshes that list from the blockchain. `domain-mint` tracks the on chain minting step of a purchase. `domain-tenure` is a newer addition that tracks how long a user has actually held a domain, for reasons explained in its own note. There is also a `freename` sub folder and a `static-data` folder sitting alongside the seven registered sub modules, both covered briefly in the last note here.

## How to read these notes

1. [01-domain-entities-what-a-domain-actually-is.md](01-domain-entities-what-a-domain-actually-is.md), covering the core entities, field by field, and what actually makes a domain here different from one you would buy on GoDaddy.
2. [02-domain-search-and-availability.md](02-domain-search-and-availability.md), covering the busiest read path in the entire API, how a name gets checked against up to ten different blockchain providers at once, and the custom in memory caching this module built for itself.
3. [03-domain-provider-and-tlds.md](03-domain-provider-and-tlds.md), covering the small lookup table that makes the whole multi provider design possible.
4. [04-domain-order-purchase-flow.md](04-domain-order-purchase-flow.md), covering what happens the moment a user clicks buy, cart validation, coupons, and Stripe.
5. [05-domain-order-blockchain-handoff-and-completion.md](05-domain-order-blockchain-handoff-and-completion.md), covering exactly where this module stops and the blockchain integration folders take over, and how a completed transaction comes back and finishes the order.
6. [06-domain-detail-refresh-and-rate-limiting.md](06-domain-detail-refresh-and-rate-limiting.md), covering the domain list a user actually owns, and the specific `refresh_domain` route that `main.ts` rate limits more tightly than almost anything else in the API.
7. [07-domain-tenure-ownership-history.md](07-domain-tenure-ownership-history.md), covering a smaller, event driven system that tracks exactly when a user first came to own a domain, built for provenance and history rather than for the checkout flow.
8. [08-domain-favorites-search-log-secondary-pieces.md](08-domain-favorites-search-log-secondary-pieces.md), covering the smaller, simpler pieces of this folder, favorites, search logging, the freename stub controller, and the static TLD list, together and briefly since none of them carry the weight the earlier files do.

## The one sentence version, if you read nothing else

A user searches a name, this module asks whichever blockchain or registrar actually issues that top level domain whether it is free, the user pays through Stripe or their own crypto wallet, this module writes a pending order and a pending mint record into Postgres, hands the actual minting transaction off to a chain specific integration module, and once that transaction confirms, writes the real owner address and blockchain explorer link back onto the domain record, at which point the user owns something that is really a token on a blockchain, not a row in a registrar's private database the way a GoDaddy domain is.
