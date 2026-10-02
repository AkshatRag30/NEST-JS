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

`recentdomains` is the one folder in this trio that does genuinely reach out to external chains and APIs, which makes it worth reading carefully rather than dismissing outright, but what it builds is a product feature, a "recently registered" ticker for the homepage, not a per domain lookup a buyer's own action would trigger. `recentDomainService` opens its own `ethers.JsonRpcProvider` (written `ethers.providers.JsonRpcProvider` before the ethers v6 upgrade in commit `71fd2fec`) for Arbitrum, BNB, and Ethereum, and separately calls Bonfida's public sales API, then walks each chain's event logs by hand, in chunks, looking specifically for `NameRegistered` events:

```ts
// src/components/recentdomains/recent-domain.service.ts (lines 166 to 180, simplified, ethers v6 form)
const eventTopic = ethers.id("NameRegistered(string,bytes32,address,uint256,uint256,uint256)");
const filter = { address: this.arbContract.target, topics: [eventTopic] };
for (let startBlock = currentBlock; startBlock > fromBlock; startBlock -= chunkSize) {
    const chunkLogs = await this.arbProvider.getLogs({ ...filter, fromBlock: chunkFrom, toBlock: startBlock });
    ...
}
```

`seedAllRecentDomains`, which is not on any `@Cron` at all but is exposed as the unguarded `GET /api/v1/recent-minted/get-all` route (no `@UseGuards`, only a Swagger `@ApiBearerAuth` label) at `src/components/recentdomains/recent-domains.controllers.ts` line 15, so it runs whenever something external calls that URL rather than per page load, pulls a fixed ratio of results from each source (eight ENS, eight Unstoppable Domains, eight Solana, three Arbitrum, three BNB), replaces the entire `tbl_recent_domain` table with that fresh batch in `RecentDoaminsMangment.addRecentDomain`, and a separate, cheap `getAllRecentMintedDomains` endpoint just reads that table back for the frontend. This is architecturally sound for what it is: expensive, multi chain log scanning happens on a schedule and gets cached in Postgres, so the actual page load is one fast database query. It is also worth noting, because it is the kind of thing worth flagging rather than assuming is fine, that `getRecentBnbDomains` hardcodes a public Infura BNB RPC URL directly in the constructor rather than reading it from config the way every other RPC URL in this same file does:

```ts
// src/components/recentdomains/recent-domain.service.ts (line 54, still hardcoded after the v6 upgrade)
this.bnbProvider = new ethers.JsonRpcProvider('https://bsc-mainnet.infura.io/v3/b1d4662319b343259d5e71620840485a');
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

## Update from the October 2026 uat pull

`custom-domain` and `fetch-domains` did not change in this pull. `recentdomains` did, substantially, because it is the heaviest user of ethers in the older part of the codebase, and commit `71fd2fec` ("Ether versoin 6") rewrote 57 lines of `src/components/recentdomains/recent-domain.service.ts` for ethers v6. The excerpts above have been corrected in place. This is the best single file in the repository for seeing what a v5 to v6 migration really involves, because almost every kind of v6 change shows up here.

The provider types and constructors at lines 16 to 19 and 53 to 57 moved from `ethers.providers.JsonRpcProvider` to `ethers.JsonRpcProvider`, and `ethers.utils.id(...)` became `ethers.id(...)` at lines 166, 202, 254 and 282. `id` is just `keccak256(toUtf8Bytes(text))`, used here both to build an event topic and, with `.slice(0, 10)`, to build a 4 byte function selector for spotting `bulkRegister` transactions.

The contract address property changed name. In v5 a `Contract` exposed `.address`. In v6 the same value lives on `.target`, typed `string | Addressable`, and `.address` simply does not exist. So `address: this.arbContract.address` became `address: this.arbContract.target` at line 168, and the same at lines 256, 326 and 332. If this had been missed, the filter would have carried `address: undefined`. The new spec's comment says `getLogs` would then "match nothing, forever", but in practice an absent address means no address filter at all, so the query would match that event topic on every contract on the chain, pulling in other registrars' `NameRegistered` events with no error. Either way it is a silent wrong answer, which is why the regression test matters.

Event filters became asynchronous. This is the subtlest change in the file:

```ts
// src/components/recentdomains/recent-domain.service.ts (lines 317 to 328)
// ethers v6: contract.filters.EventName() returns a DeferredTopicFilter, not the
// plain { topics } object v5 returned — the actual topics array must be awaited
// via .getTopicFilter() before it can be passed to provider.getLogs.
const registerTopics = await this.ensContract.filters.NameRegistered().getTopicFilter();
const renewTopics = await this.ensContract.filters.NameRenewed().getTopicFilter();
const network = await this.ethProvider.getNetwork(); // Get chain ID
const logsRegistered = await this.ethProvider.getLogs({
    fromBlock,
    toBlock: currentBlock,
    address: this.ensContract.target,
    topics: registerTopics,
});
```

In v5 the code was `const registerFilter = this.ensContract.filters.NameRegistered();` followed by `topics: registerFilter.topics`. In v6 that object has no `.topics` field, so the old code would have passed `topics: undefined` and fetched every log the ENS controller emitted, not just registrations.

Chain ids became `bigint`. In v6 `(await provider.getNetwork()).chainId` is a `bigint` such as `137n`, not the number `137`. That broke two things silently. `testUdNetwork` at line 153 used to be `return network.chainId === 137;`, and since `137n === 137` is `false` in JavaScript (strict equality never converts between `bigint` and `number`), every UD fetch would have thrown "UD Network connection failed" forever. It now reads `return Number(network.chainId) === 137;`. And `getChainName(chainId)` looks the id up in a `Record<number, string>`, where `chainMap[1n]` actually does work by accident (object keys are strings, and `String(1n)` is `"1"`), but the parameter is typed `number`, so the four call sites at lines 126, 222, 291 and 427 now wrap it in `Number(...)`. `block.timestamp` stayed a plain number in v6, so the `.sort((a, b) => b.timestamp - a.timestamp)` calls are safe.

There are several honest risks that remain in this file after the migration, most of them older bugs that the migration did not touch.

First, `parseLog` changed contract. In v5, `Interface.parseLog` threw for a log whose topic did not match any event in the ABI. In v6 it returns `null` instead. The BNB path handles that correctly with `if (parsed && parsed.name === 'NameRegistered')` at line 295. The Arbitrum path at line 219 does not check, but it sits inside a `try` that returns `null`, so a `null` parse only drops that one log. `parseLogWithTimestamp` at line 120 does not check either and has no `try`, so if a non matching log ever reached it, `parsed.args` would throw `TypeError: Cannot read properties of null` and reject the whole `Promise.all` for ENS. Today the ENS logs are pre filtered by topic, so this is latent rather than live.

Second, `getRecentUdDomains` uses `this.udContract.queryFilter(newUriFilter, ...)` at line 400. In v6 that returns `Array<EventLog | Log>`, and only an `EventLog` has `.args`. A log the ABI could not decode comes back as a plain `Log`, and `log.args.uri` at line 422 then throws, which escapes the loop and makes the whole UD feed fail.

Third, an older bug the new spec does not cover: `getRecentBnbDomains` (lines 246 to 313) retries three times and, if every attempt throws, falls off the end of the loop and returns `undefined`. `seedAllRecentDomains` wraps it in `safeFetch`, which only converts thrown errors into `[]`, not `undefined` results, so `...bnbDomains.slice(0, 3)` at line 465 then throws `TypeError: Cannot read properties of undefined (reading 'slice')`, the whole seed request fails, and `tbl_recent_domain` is not refreshed at all, even though ENS, UD and Solana succeeded. A `return [];` after the loop would fix it.

Fourth, the providers. Four `JsonRpcProvider` objects are created in the constructor without the `staticNetwork` option that the marketplace v2 code uses (see `src/components/marketplacev2/order/order.service.ts` lines 154 to 159). Because these are long lived, the v6 behaviour of retrying network detection once a second for an unreachable URL applies directly, so a dead `POL_RPC_URL` produces a log line every second for the life of the process once the first UD request is made. Line 56 also builds a provider for `ETH_URL` and line 57 immediately overwrites it, a wasted object.

Fifth, v6 changes what happens when a contract address config value is bad. The new spec's fixture comment at `recent-domain.service.spec.ts` lines 14 to 19 records a real discovery: the v6 `Contract` constructor treats any target string that is not a hex address as an ENS name and eagerly calls `provider.resolveName(...)` on it, which crashed every test with an unhandled `ECONNREFUSED` until the fixtures were changed. The constructor here never validates `BNB_CONTRACT_ADD`, `ENS_CONTRACT_ADD`, `ARB_CONTRACT_ADD` or `UD_CONTRACT_ADD` (line 49 only checks the three RPC URLs), so a typo in Secrets Manager no longer fails as a simple bad address, it becomes a background ENS lookup against that chain's RPC, and a missing value is passed straight to `new ethers.Contract(undefined, ...)`. Both cases are worth checking against a local run before relying on them.

The hardcoded Infura key on line 54 is unchanged by the migration and is still a live credential in source.

The spec file `src/components/recentdomains/recent-domain.service.spec.ts` existed before this pull with two Bonfida tests, and commit `71fd2fec` grew it to seven tests. The fixture now uses real looking hex addresses (`0x1111...` to `0x4444...`) for the four contract addresses, for the reason explained above. The five new tests are `testUdNetwork` returning `true` for `{ chainId: 137n }` and `false` for `{ chainId: 56n }` (the `bigint` comparison regression), `getChainName(Number(137n))` returning `'Polygon'`, `getRecentArbDomains` calling `getLogs` with `expect.objectContaining({ address: '0xArbTargetAddress' })` after swapping in a fake contract whose `target` is that string (the `.target` rename regression), and `parseLogWithTimestamp` resolving `'Ethereum'` from a `bigint` `chainId: 1n`. Every chain object is replaced by hand on the service instance (`(service as any).udProvider = { getNetwork: jest.fn()... }`), so these tests verify the v6 call shapes without ever touching a node. For the newer, purpose built way this codebase now reads chain events, see [../12-marketplace-v2/08-the-onchain-event-poller-architecture.md](../12-marketplace-v2/08-the-onchain-event-poller-architecture.md), and for the whole migration see [../12-marketplace-v2/12-ethers-v6-migration-and-the-new-test-suite.md](../12-marketplace-v2/12-ethers-v6-migration-and-the-new-test-suite.md).
