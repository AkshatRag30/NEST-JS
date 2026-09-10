# 08. The Smaller Pieces, Covered Together

This module has a handful of sub folders that are genuinely simple, and are worth understanding at a glance rather than in the same depth as the search, order, detail, and tenure pieces, since none of them carry anything close to the same weight.

## `domain-favorite`, a watch list

`DomainFavoriteEntity` (`tbl_fav_domain`) is four columns, a domain name, its provider, a user id, and a unique identifier built from combining the two so the same user cannot favorite the same domain twice. `DomainFavoriteService` is a thin wrapper around a repository, create, read all by user, delete one, delete several by id in bulk. The one place this sub module gets called from outside itself is worth remembering, from `DomainOrderService.create`, right after an order is saved, which calls `clearFavDomainByDomainProviderAndUserId` to automatically drop any favorited domains from that same provider once the user has actually bought them.

```ts
// domain-favorite.service.ts
async clearFavDomainByDomainProviderAndUserId(userId: string, domainProvider: string): Promise<void> {
    await this.domainFavoriteRepo.bulkRemoveFavDomainByDomainProviderAndUserId(userId, domainProvider);
}
```

## `domain-search-log`, quiet analytics

`DomainSearchLogEntity` (`tbl_domain_search_log`) records `domainName`, `tld`, `isDomainAvailable`, and `isValidSearch` for literally every search this company's users make, whether they are signed in or not, indexed on created date, TLD, domain name, and both boolean flags to keep the analytics queries built on top of it fast. As the search note already covered, every write into this table happens fired off in the background from `DomainSearchService`, never blocking the actual search response the user is waiting on. `DomainSearchLogService` (the class backing this table, injected into search as `DomainSearchLogServiceInterface`) is what ultimately answers the `domain/search/analytics`, `searched_names`, and `searched_tld` endpoints, letting the business see aggregate demand across every search ever made, independent of whether that search ever turned into an account or a sale.

## `freename`, a stub controller for a registrar this module barely touches directly

```ts
// src/components/domain/freename/freename.controller.ts
@Controller('freename/search')
export class FreenameSearchController {
    constructor(@Inject('IFreenameSearchServiceInterface') private readonly searchService: IFreenameSearchServiceInterface) {}
    @Get()
    async searchDomains(@Query('search') search: string): Promise<any> {
        return this.searchService.searchDomains(search);
    }
}
```

This is a small, standalone controller sitting inside the `domain` folder but really acting as the front door to the dedicated `freename` integration component (covered by another agent's notes), a single unauthenticated search endpoint with no caching or logging of its own. The heavier Freename logic that actually matters to this module, checking availability and pricing for a Freename domain during a real search or suggestion call, is injected into `DomainSearchService` directly as `FreenameIntegrationServiceInterface`, this controller is a separate, thinner surface area, worth noticing as an example of a provider having its logic split across more than one place in the codebase rather than living in one single tidy folder.

## `static-data`, a hardcoded map for a specific narrow purpose

```ts
// src/components/domain/static-data/tld-list.array.ts
export const TLDList = [
    { domainProvider: DomainProvider.UNSTOPPABLE_DOMAINS, domainList: ['crypto', 'nft', 'x', 'wallet', 'bitcoin', '888', 'blockchain', 'zil', 'dao', 'polygon'] },
    { domainProvider: DomainProvider.ENS, domainList: ['eth'] },
    { domainProvider: DomainProvider.Bonfida, domainList: ['sol'] },
    { domainProvider: DomainProvider.Tezos, domainList: ['tez'] },
    { domainProvider: DomainProvider.Arbitrum, domainList: ['arb'] },
    { domainProvider: DomainProvider.BinanceSmartChain, domainList: ['bnb'] }
];
```

This is a hardcoded, in code duplicate of the same provider to TLD relationship the database already stores properly in `tbl_domain_provider_and_tlds` (covered in note three). It exists as its own plain array file rather than a database read, almost certainly because whatever consumes it needs that mapping synchronously, at import time, with zero possibility of a database round trip or a cache miss, worth flagging as exactly the kind of small, deliberate duplication a large, real codebase accumulates over time between a fast, static, code level source of truth and the flexible, editable database table used everywhere else, rather than something to "clean up" without first checking every place that actually depends on it staying fast and synchronous.
