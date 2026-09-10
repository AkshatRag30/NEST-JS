# 09. Ads and Web3 Advertising

## Not a Google or Meta ad integration, an ad system for this company's own product

The natural first guess for a folder named `ads` is a wrapper around a third party advertising platform, buying placements on Google or Meta. That is not what this is. Every piece of this folder ties directly into `@components/web3/ipfs`, this company's own module for hosting the actual decentralized websites its customers publish at their owned Web3 domains. What `ads` manages is Endless Domains' own ad inventory shown directly on those customer owned sites, banners that appear on a page someone reaches by visiting a `.eth` or similar domain the company sold, tracked by the specific IPFS hash and domain name of the page a visitor was looking at.

## The entity's shape hints at more than the current endpoints use

```ts
@Entity({ name: 'tbl_ads' })
export class AdsEntity extends BaseEntity {
    @Column({ nullable: false, name: 'name', unique: true }) name: string;
    @Column({ nullable: true, name: 'image_url' }) image_url: string;
    @Column({ nullable: false, name: 'type' }) type: string;
    @Column({ nullable: true, name: 'user_id' }) user_id: string;
    @Column({ nullable: true, name: 'domain_provider' }) domain_provider: string;
    @Column({ nullable: true, name: 'domain_name' }) domain_name: string;
    @Column({ nullable: true, name: 'redirect_url' }) redirect_url: string;
    @Column({ nullable: true }) position: string;
    @Column({ nullable: true }) status: string;
}
```

`user_id`, `domain_provider`, `domain_name`, `position`, and `status` all sit on the entity, but `CreateAdsDto` only actually accepts `name`, `image_url`, `type`, and an optional `redirect_url`, none of the create or update paths in this cluster's version of `AdsService` ever populate the other five columns. That gap between what the table can hold and what the current API actually writes usually means one of two things, either a targeting feature (showing a specific ad only on a specific domain provider's pages, or only to a specific user) was planned and partially built before this cluster's slice of the code was finished, or those columns get written by some other part of the system entirely, outside what this cluster covers. Either way, it is worth treating the wider column list as a hint about intended future capability rather than assuming the current four-field create form is the whole feature.

## The public facing loop: showing an ad and recording that it was seen

`GET /ads` (`checkIpfsExists`) is how a hosted Web3 site actually asks this backend which ads to show, it takes an `ipfsHash` and a `domainName`, confirms through `IpfsService.checkIfIpfsOrDomainExists` that the combination is real, and if so returns every ad currently in the system. `POST /ads/ads_viewed` is the other half of that loop, called by the visitor's own browser once an ad has actually been rendered on the page, recording the `ipfsHash`, `domainName`, the visitor's IP address (read directly off the request, `req.ip`), and which specific ad was shown into a separate log table, `IpfsLogsEntity`, that lives in the `web3/ipfs` module rather than here. Neither of these two routes carries a `@UseGuards(...)` decorator, which makes sense for `ads_viewed` (it has to be reachable by any anonymous visitor's browser to do its job at all) and is more debatable for `checkIpfsExists`, which is documented with `@ApiBearerAuth('defaultBearerAuth')` in its Swagger annotation but has no guard actually enforcing that a bearer token be present.

Creating, updating, and deleting an ad (`POST`, `PUT`, `DELETE /ads`) all require `AccessTokenGuard`, and the two reporting routes, `GET /ads/ads_viewed/analytics` and `GET /ads/details/:id`, require it as well, so the actual administrative surface of this feature is consistently protected even where the public read side is looser.

## A real, precise bug in the view analytics filter

`AdsRepo.findAdsViewedData` supports filtering the ad view log by an `ALL`, `DOMAINPROVIDER`, or `SEARCH` mode. The `SEARCH` branch is where the bug sits:

```ts
} else if (filterBy == 'SEARCH') {
    const totalNoOfRows = await mainQuery
        .where('ads_viewed."domainName" like :domainName', { domainName: `%${search.toLowerCase}%` })
        .orWhere('ads_viewed."domainName" like :domainName', { domainName: `%${search}%` })
        .getRawMany();
    ...
```

`search.toLowerCase` is missing its call parentheses, `()`. As written, this does not lowercase the search term at all, it takes the `toLowerCase` function itself, and JavaScript's string interpolation then coerces that function into its own source code text inside the query parameter, producing a nonsensical `LIKE` pattern that can never genuinely match a real domain name. The very next line in the same query, the `orWhere` using the untouched `search` value directly, happens to still work correctly, which is almost certainly why this defect has not been noticed, the search feature still returns correct results for anyone searching with the exact casing already stored in the database, it just never gets the case-insensitive fallback the first clause was clearly meant to provide. This is exactly the kind of small, precise, easy to miss defect worth pointing at directly rather than only describing in the abstract, since it is fully visible and confirmable by reading these two lines side by side.

## No caching on this reporting endpoint either

`GET /ads/ads_viewed/analytics` runs a filtered, paginated join across `tbl_ads` and the IPFS view log on every single call, exactly the shape of endpoint the root `app.module.ts` comment about `CacheModule` describes as the intended target for `CacheInterceptor`, and yet no caching decorator appears anywhere on this controller. Combined with the equivalent gaps already noted in [01-admin-user-management.md](01-admin-user-management.md) and parts of [08-internal-search-analytics-and-event-tracking.md](08-internal-search-analytics-and-event-tracking.md), the honest overall picture for this cluster is that the caching convention is real and correctly used in the newer Google Analytics and Search Console work, but was not consistently carried into every dashboard style endpoint written elsewhere in the same codebase.
