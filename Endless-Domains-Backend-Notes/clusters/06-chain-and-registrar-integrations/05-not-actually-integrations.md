# 05. Three folders that live here but are not really integrations

The task behind this note was explicit about not assuming a folder belongs in this cluster just because it sits next to the others and has "domain" in its name. `custom-domain`, `recentdomains`, and `fetch-domains` were each read completely with that question in mind, and each one turns out to be something else, for a different reason.

## custom-domain: a curated catalog, not a live lookup

`custom-domain` has no outbound HTTP call anywhere in it. What it actually is becomes clear from its entity and its one distinctive method. `CustomDomainEntity` is a plain Postgres row: a domain name, a price stored as a string, a status (`available`, `sold`, `returned`), a category, an event name, and an image URL, all set once by an admin and read back later. The method that makes its real purpose obvious is `seedDomainsFromS3Excel`, which downloads an Excel file from S3, and for every row inserts a `CustomDomainEntity` tagged with a hardcoded `event: 'Chinese New Year'` and a hardcoded Unstoppable Domains logo URL:

```ts
const payload: CreateCustomDomainRecordDto = {
    domainName: row.domainName,
    price: row.price,
    category: row.category,
    event: 'Chinese New Year',
    domainProvider: DomainProvider.UNSTOPPABLE_DOMAINS,
    eventImage: 'https://edimages.s3.us-east-1.amazonaws.com/prod/domain_providers/UD/UD.svg',
};
```

This is a marketing feature: a hand curated list of premium or themed domain names an admin has already secured on some registrar, listed for sale on the site under a promotional banner, with its own filterable, paginated `getCustomDomains` endpoint for the storefront to browse. The `domainProvider` and `network` fields on each row exist purely so the frontend can show the right chain logo next to a listing, not because this service ever asks that registrar anything live. It sits in this cluster because its data happens to describe domains that live on the same chains and registrars the rest of this cluster integrates with, not because it integrates with anything itself.

## recentdomains: a homepage activity ticker, built from real chain reads

`recentdomains` is the one folder in this trio that does genuinely reach out to external chains and APIs, which makes it worth reading carefully rather than dismissing outright, but what it builds is a product feature, a "recently registered" ticker for the homepage, not a per domain lookup a buyer's own action would trigger. `recentDomainService` opens its own `ethers.providers.JsonRpcProvider` for Arbitrum, BNB, and Ethereum, and separately calls Bonfida's public sales API, then walks each chain's event logs by hand, in chunks, looking specifically for `NameRegistered` events:

```ts
const eventTopic = ethers.utils.id("NameRegistered(string,bytes32,address,uint256,uint256,uint256)");
const filter = { address: this.arbContract.address, topics: [eventTopic] };
for (let startBlock = currentBlock; startBlock > fromBlock; startBlock -= chunkSize) {
    const chunkLogs = await this.arbProvider.getLogs({ ...filter, fromBlock: chunkFrom, toBlock: startBlock });
    ...
}
```

`seedAllRecentDomains`, run on some external schedule rather than per request, pulls a fixed ratio of results from each source (eight ENS, eight Unstoppable Domains, eight Solana, three Arbitrum, three BNB), replaces the entire `tbl_recent_domain` table with that fresh batch in `RecentDoaminsMangment.addRecentDomain`, and a separate, cheap `getAllRecentMintedDomains` endpoint just reads that table back for the frontend. This is architecturally sound for what it is: expensive, multi chain log scanning happens on a schedule and gets cached in Postgres, so the actual page load is one fast database query. It is also worth noting, because it is the kind of thing worth flagging rather than assuming is fine, that `getRecentBnbDomains` hardcodes a public Infura BNB RPC URL directly in the constructor rather than reading it from config the way every other RPC URL in this same file does:

```ts
this.bnbProvider = new ethers.providers.JsonRpcProvider('https://bsc-mainnet.infura.io/v3/b1d4662319b343259d5e71620840485a');
```

That is a real, live Infura project id committed straight into source, not a placeholder, sitting right next to a `BNB_URL` config value the constructor already reads and simply does not use for this one provider.

## fetch-domains: an internal event handler wearing an integration's clothes

`fetch-domains` is the smallest of the three, and the easiest to see through once you notice what it actually depends on. `FetchDomainService` has exactly one method, and it is wired to an internal application event, not a route:

```ts
@OnEvent('web3UserLogin.UpdateDomainDetailTable', { async: true })
async updateDomainDetialTable(payload: UpdateDomainDetailTableAfterLoginEvent): Promise<void> {
```

When a user logs in with a wallet, this handler fires, looks up that user's wallet addresses, and calls out to five separate Alchemy wrapper services (`ArbAlchemyServiceInterface`, `UdBaseAlchemyServiceInterface`, `EnsAlchemyServiceInterface`, `UdAlchemyServiceInterface`, `BnbAlchemyServiceInterface`, `FreenameAlchemyServiceInterface`) plus a Moralis lookup for Solana, to find every domain NFT that wallet actually owns, then bulk writes the results into the domain detail table and emits a `domain.sync.completed` event when it is done. Every one of the calls doing real work here belongs to the `@components/alchmey` folder, a different cluster entirely, not to anything inside this one. `fetch-domains` itself contributes no HTTP call, no chain read, and no registrar logic of its own, it is purely an orchestrator that reacts to a login event and delegates everything to services this cluster does not own. It sits here almost by association: its job is to fetch which domains a user owns across chains, which sounds like it belongs next to `recentdomains`, but the actual integration work happens entirely somewhere else.

## Why this distinction is worth making at all

A folder's name and its location in the source tree are both real signals about what a team intended, but neither one is a guarantee about what the code inside actually does, and all three of these are proof of that in slightly different ways: one is a marketing catalog with domain flavored fields, one is a real multi chain integration wearing the shape of a caching layer, and one is an event handler that reads as an integration because of what it triggers, not because of anything it does itself.
