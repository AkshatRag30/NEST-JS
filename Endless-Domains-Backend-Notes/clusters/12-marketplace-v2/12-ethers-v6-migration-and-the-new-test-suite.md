# 12. The ethers v6 Migration and the Arrival of a Real Test Suite

## Why this note sits in the marketplace v2 cluster

Two things landed on the `uat` branch in September 2026 that touch far more than marketplace v2, but were done because of it. The first is a major version upgrade of `ethers`, the JavaScript library this backend uses to build, hash and verify blockchain transactions, from version 5 to version 6. Marketplace v2 is written against ethers v6 from the ground up (its Seaport order builder, its event poller, its wallet verification), and the two major versions cannot comfortably coexist under the same package name, so every older module that used ethers had to move too. The second is a large batch of new Jest spec files. Marketplace v2 arrived with fifty of them, and the migration work added another dozen around the older modules it touched, specifically to prove that nothing changed behaviour when the library underneath them did.

This note is the single place that explains both, across the whole repository, so that the per module notes in clusters 03, 04, 05, 06, 08 and 10 can stay focused on their own features and just link here. If you are a frontend developer who has used ethers in a dapp, a lot of this will feel familiar, because the browser side of the ecosystem went through exactly the same migration.

## What actually changed in `package.json`

Commit `71fd2fec` ("Ether versoin 6", 11 September 2026) made the one line change that started everything:

```json
// package.json (dependencies, line 70 on uat)
"ethers": "^6.13.4",
```

It used to read `"ethers": "^5.7.2"`. `package-lock.json` now resolves that range to ethers `6.17.0` (lockfile line 12164), whose own dependencies are a completely different set from v5's, `@noble/curves`, `@noble/hashes`, `@adraffy/ens-normalize`, `aes-js` and `tslib`, rather than the two dozen `@ethersproject/*` packages that made up v5. The same commit migrated most of the code. Commit `26b1f0e4` ("Upgraded the ethers version", 14 September 2026) finished the ENS integration and the two renewal modules and added several guard tests. Commit `2c7fca78` ("Completed the B03 sprint and tested the apis", 16 September 2026) fixed the one plain JavaScript script, `src/scripts/verify-nft-template.js`. In parallel, marketplace v2's own commits (starting with `836d5f89` on 10 September 2026) were written in v6 style from the first line.

`web3` and `web3-utils` stayed on `^1.10.0`. So the split described in [../05-blockchain-infrastructure/02-web3js-and-ethersjs-two-libraries-one-job.md](../05-blockchain-infrastructure/02-web3js-and-ethersjs-two-libraries-one-job.md), web3.js mostly for reads and ethers for building and verifying transactions, is still how the codebase is organised. Only the ethers half changed dialect.

## The three ideas behind almost every change

Before the table, it helps to hold three ideas in your head, because nearly every one of the forty or so changed lines in this repository is an instance of one of them.

The first idea is that v6 flattened the namespaces. In v5 you reached most helpers through `ethers.utils.something` and providers through `ethers.providers.Something`. v6 deleted both namespaces and exports everything at the top level, so `ethers.utils.keccak256` is now just `ethers.keccak256`, and `ethers.providers.JsonRpcProvider` is now `ethers.JsonRpcProvider`. You can also import them by name, `import { keccak256, JsonRpcProvider } from 'ethers'`, which is the style marketplace v2 uses. The algorithms behind these functions did not change, keccak256 and ABI encoding produce the same bytes in both versions. This is a pure rename, but because the old names no longer exist, missing even one turns into a compile error or a `Cannot read properties of undefined (reading 'keccak256')` at runtime.

The second idea is that `BigNumber` is gone, replaced by native `bigint`. Every on chain integer in v5 came back as a `BigNumber` object with methods like `.toNumber()`, `.isZero()`, `.eq()` and `.add()`. In v6 the same values come back as the JavaScript primitive `bigint`, written in code as `42n`. That is a real improvement, you get exact arithmetic with ordinary operators (`a + b`, `a === 0n`), but it has three sharp edges. A `bigint` has no `.toNumber()`. A `bigint` is never strictly equal to a `number` (`137n === 137` is `false`). And `JSON.stringify` throws `TypeError: Do not know how to serialize a BigInt` if it meets one, which matters a great deal in a NestJS app where every controller return value is serialised to JSON.

The third idea is that the provider and contract APIs were reshaped in a few specific places. A `Contract`'s address moved from `.address` to `.target`. `contract.filters.EventName()` now returns a deferred object whose topics you must `await` with `.getTopicFilter()`. `Interface.parseLog` returns `null` for a log it does not recognise instead of throwing. `provider.getSigner()` became asynchronous. `provider.getBlock()` is typed to return `Block | null`. `network.chainId` is a `bigint`. And a `JsonRpcProvider` that cannot reach its node keeps retrying network detection in the background rather than failing fast, unless you give it a `staticNetwork` hint. Each of these shows up somewhere in this codebase, and each is walked through below.

## The translation table, built from the real diffs

Every row below is a change that actually exists in this repository between the baseline commit `a131b429` and `uat` HEAD, with the file and line on `uat` where you can see the v6 form.

| ethers v5 form | ethers v6 form | Where it changed on `uat` |
|---|---|---|
| `new ethers.utils.Interface(abi)` | `new ethers.Interface(abi)` | `contract-deployment.service.ts:63`, `nft-collection.service.ts:67`, `eth-domain-renewal.service.ts:176`, `bnb-arb-domain-renewal.service.ts:228`, `gm-verification.service.ts:270`, `marketplace-event-decoder.ts:4`, `verify-nft-template.js:21` |
| `ethers.utils.keccak256(data)` | `ethers.keccak256(data)` | `deployment-verification.service.ts:103`, `contract-deployment.service.ts:68`, `nft-collection.service.ts:72`, `eth-domain-renewal.service.ts:320`, `bnb-arb-domain-renewal.service.ts:383`, `label-hash.util.ts:5` |
| `ethers.utils.toUtf8Bytes(s)` | `ethers.toUtf8Bytes(s)` | `label-hash.util.ts:5` |
| `ethers.utils.id(text)` | `ethers.id(text)` | `gm-verification.service.ts:80`, `recent-domain.service.ts:166, 202, 254, 282` |
| `ethers.utils.namehash(name)` | `ethers.namehash(name)` | `bnb-arb-domain-renewal.service.ts:179` |
| `ethers.utils.getAddress(addr)` | `ethers.getAddress(addr)` | `deployment-verification.service.ts:142`, `contract-deployment.service.ts:43, 51`, `nft-collection.service.ts:58`, `eth-domain-renewal.service.ts:292`, `bnb-arb-domain-renewal.service.ts:355`, `gm.service.ts:330`, `web3-auth.service.ts:64, 83, 146` |
| `ethers.utils.verifyMessage(msg, sig)` | `ethers.verifyMessage(msg, sig)` | `web3-auth.service.ts:124` |
| `ethers.utils.parseUnits(v, d)` returning `BigNumber` | `ethers.parseUnits(v, d)` returning `bigint` | `contract-deployment.service.ts:61` |
| `ethers.utils.formatUnits(bn, d)` | `ethers.formatUnits(big, d)` | `marketplace-event-decoder.ts:42` |
| `ethers.utils.LogDescription` (type) | `ethers.LogDescription` | `marketplace-event-decoder.ts:15, 16, 36, 64, 94` |
| `new ethers.providers.JsonRpcProvider(url)` | `new ethers.JsonRpcProvider(url)` | `deployment-verification.service.ts:71`, `contract-deployment.service.ts:278`, `nft-collection.service.ts:313`, `gm-verification.service.ts:178`, `recent-domain.service.ts:53 to 57` |
| `ethers.providers.TransactionResponse`, `TransactionReceipt`, `Block` (types) | `ethers.TransactionResponse`, `ethers.TransactionReceipt`, `ethers.Block` | `deployment-verification.service.ts:68, 110, 158`, `gm-verification.service.ts:181, 199, 315` |
| `new ethers.providers.Web3Provider(eip1193)` | `new ethers.BrowserProvider(eip1193)` | `verify-nft-template.js:16` |
| `provider.getSigner(0)` (sync) | `await provider.getSigner(0)` | `verify-nft-template.js:17` |
| `BigNumber.from(x)` then `.isZero()` | `BigInt(x)` then `=== 0n` | `listing-id.util.ts:10, 14` |
| `normalizeUsdPrice(p: ethers.BigNumber)` | `normalizeUsdPrice(p: bigint)` | `marketplace-event-decoder.ts:41` |
| `estimate.toNumber()` | `Number(estimate)` | `contract-deployment.service.ts:280`, `nft-collection.service.ts:315` |
| `network.chainId === 137` (number) | `Number(network.chainId) === 137` (bigint in v6) | `recent-domain.service.ts:153`, plus `Number(...)` at 126, 222, 291, 427 |
| `contract.address` | `contract.target` | `recent-domain.service.ts:168, 256, 326, 332` |
| `contract.filters.X().topics` | `await contract.filters.X().getTopicFilter()` | `recent-domain.service.ts:320, 321` |
| `iface.parseLog(log)` throws on unknown topic | returns `null`, must be checked | `marketplace-event-decoder.ts:22 to 25` |
| `ethers.Wallet.createRandom(['i'])` | `ethers.Wallet.createRandom()` | `ens-integration.service.ts:227` |
| `import { providers } from 'ethers'` (unused) | removed | `refund-repo.service.ts` (old line 12) |
| no equivalent | `new JsonRpcProvider(url, chainId, { staticNetwork: true })` | marketplace v2 only, `order.service.ts:159`, `chain-event-source-poller.service.ts:97`, both poller application services |
| `ethers.BigNumber.from('42')` in tests | `42n` | `marketplace-event-decoder.spec.ts` throughout |

## The changes that are not just renames

Most rows above are mechanical. Six are worth walking through properly, because each one would have caused a real, silent production bug if it had been missed, and that is exactly the kind of thing a code review of a "just bump the version" pull request misses.

The first is `.toNumber()`. Both deployment services estimate gas like this now:

```ts
// src/components/contract-deployment/services/contract-deployment.service.ts (lines 278 to 280)
const provider = new ethers.JsonRpcProvider(rpcUrl);
const estimate = await provider.estimateGas({ data, from: fromAddress });
return Number(estimate);
```

With the old `estimate.toNumber()`, v6 would have thrown `TypeError: estimate.toNumber is not a function`, and because that line sits inside a `try` whose `catch` returns `DEFAULT_GAS_ESTIMATE_FALLBACK`, nobody would have noticed, every prepare call would simply have returned the fallback forever. `Number(estimate)` is correct and safe, since no gas value comes anywhere near `Number.MAX_SAFE_INTEGER`. It also matters that the result is a plain number, because `gasEstimate` is returned in the JSON body, and a raw `bigint` there would make the whole response fail to serialise.

The second is the chain id comparison in the homepage ticker:

```ts
// src/components/recentdomains/recent-domain.service.ts (lines 150 to 157)
async testUdNetwork(): Promise<boolean> {
    try {
        const network = await this.udProvider.getNetwork();
        return Number(network.chainId) === 137; // Expect Polygon mainnet
    } catch (error) {
        return false;
    }
}
```

In v6 `network.chainId` is `137n`. Strict equality never converts between `bigint` and `number`, so the old `network.chainId === 137` would have been `false` on every call, and `getRecentUdDomains` would have thrown "UD Network connection failed" forever, quietly emptying the Unstoppable Domains share of the ticker. Loose equality (`137n == 137`) would actually be `true`, but this codebase, rightly, does not rely on `==`.

The third is the `parseLog` contract change:

```ts
// src/components/marketplace/abi/marketplace-event-decoder.ts (lines 18 to 28)
try {
  // ethers v6's Interface.parseLog returns null for a non-matching topic instead of
  // throwing (v5 threw here) — must be filtered explicitly, or a null slips into the
  // decoded array and callers like findMarketplaceEvent would throw when reading .name.
  const parsed = marketplaceInterface.parseLog(log);
  if (parsed) {
    decoded.push(parsed);
  }
} catch {
  continue;
}
```

A sale receipt contains logs from several contracts. In v5 every foreign log threw and was skipped. In v6 they come back as `null`, so without the `if (parsed)` guard the decoded list would have been full of `null`s, `findMarketplaceEvent` would have crashed reading `.name`, and every v1 marketplace buy confirmation in the per minute cron would have failed. The other `parseLog` call sites are safe for different reasons: `gm-verification.service.ts` line 274 sits inside a `try`, `recent-domain.service.ts` line 295 already checks `parsed &&`, and line 219 is inside a `try` that returns `null`. Only `parseLogWithTimestamp` at `recent-domain.service.ts` line 120 has no guard at all, which is latent today because its logs are pre filtered by topic.

The fourth and fifth are both in `recent-domain.service.ts` and both about contracts. `.address` became `.target`, and the old name does not exist on a v6 `Contract`, so `{ address: this.arbContract.address }` would have built a log filter with no address, matching that event topic on every contract on the chain. And `contract.filters.NameRegistered()` no longer returns `{ topics }`, it returns a `DeferredTopicFilter`:

```ts
// src/components/recentdomains/recent-domain.service.ts (lines 320 to 321)
const registerTopics = await this.ensContract.filters.NameRegistered().getTopicFilter();
const renewTopics = await this.ensContract.filters.NameRenewed().getTopicFilter();
```

The reason is that v6 contracts can resolve ENS names and addressable objects lazily, so building the final topic array may need an `await`. The old `registerFilter.topics` would have been `undefined`, fetching every ENS controller log instead of just registrations.

The sixth is the listing id guard in the v1 marketplace, which stopped using any ethers type at all:

```ts
// src/components/marketplace/abi/listing-id.util.ts (lines 7 to 17)
export function assertNonZeroListingId(listingId: string): void {
  let value: bigint;
  try {
    value = BigInt(listingId);
  } catch {
    throw new BadRequestException('Invalid listingId');
  }
  if (value === 0n) {
    throw new BadRequestException('listingId 0 is not a valid listing');
  }
}
```

This is a good example of the v6 philosophy: once integers are native `bigint`s, you often do not need the library at all. One small behavioural difference exists. `BigInt('')` returns `0n` instead of throwing, so an empty listing id now gets the "listingId 0" message instead of "Invalid listingId". Both are 400 responses, so nothing unsafe gets through.

## What deliberately stayed on v5

Three files still import from the v5 family, on purpose, and new tests pin that decision:

```ts
// src/components/alchmey/arbAlchmey/arb-alchmey.servers.ts (line 11)
import { BigNumber } from "@ethersproject/bignumber";
```

```ts
// src/components/alchmey/fetch-expiry/fetch-expiry.service.ts (line 11), same import in src/components/ud-integration/ud-integration.service.ts (line 27)
import { arrayify } from '@ethersproject/bytes';
```

These are the standalone v5 packages, not `ethers` itself, so the version bump did not touch them. `alchemy-sdk` 3.x is built on those same v5 packages, so passing it a v5 `BigNumber` token id (`arb-alchmey.servers.ts` line 96) is the correct choice, and `arb-alchmey.servers.spec.ts` asserts the argument really is a `BigNumber` and not a `bigint`. `keccak256-cross-agreement.spec.ts` proves the two private `keccak256` helpers built on `arrayify` still agree with each other.

The real risk here is packaging. Neither `@ethersproject/bignumber` nor `@ethersproject/bytes` is declared in `package.json`. Under v5 they arrived as dependencies of `ethers` itself, which is presumably why nobody ever added them. Under v6 they only arrive because `alchemy-sdk` lists them (`package-lock.json` lines 8713 to 8722) and npm hoists them to the top of `node_modules`. This is what is called a phantom dependency, code that works only by accident of the install layout. If `alchemy-sdk` is upgraded to a release built on ethers v6, or a future npm or pnpm install stops hoisting them, the app will fail at boot with "Cannot find module '@ethersproject/bignumber'". Adding both packages explicitly to `dependencies` costs nothing.

## Leftover v5 patterns found in the whole `src` tree

A search of every file under `src`, marketplace v2 included, for `ethers.utils`, `ethers.providers`, `ethers.BigNumber`, `ethers.constants`, `BigNumber`, `.toNumber()` on ethers values, `.isZero()`, `Web3Provider`, `defaultAbiCoder`, `formatBytes32String`, `solidityKeccak256`, `hexZeroPad`, `splitSignature` and `contract.address` finds no live v5 code left, apart from the intentional `@ethersproject/*` imports above. What does remain is stale text in comments, which is harmless but misleading:

| File and line | What it says |
|---|---|
| `src/@core/utils/helper.ts:47` | comment mentioning `ethers.utils.verifyMessage` (the code now calls `ethers.verifyMessage`) |
| `src/components/contract-deployment/constants/oz-erc20-template.constant.ts:5` | doc comment mentioning `ethers.utils.Interface.encodeDeploy` |
| `src/components/nft-collection/constants/oz-erc721-template.constant.ts:5` | same doc comment |
| `src/scripts/write-erc20-template-constant.js:16` | generator that writes that same v5 phrase into the constant file every time it runs |
| `src/scripts/write-nft-template-constant.js:16` | same, for the NFT constant |
| `src/components/marketplace/abi/marketplace-events.abi.ts:24` | comment mentioning `ethers.utils.parseUnits(price, 6)` in the frontend's `DomainListingModal.tsx` |

The `.contract.address` matches in the `alchmey` services (for example `arb-alchmey.servers.ts` lines 119 to 141) are not ethers at all, they are fields on Alchemy SDK response objects, so they are correct as written. The `.toNumber()` search found zero matches anywhere in `src`.

## An honest audit of `bigint` risks

Because `bigint` versus `number` mistakes are the most dangerous class of v6 bug, the older modules were checked for three specific shapes.

Comparisons between a `bigint` and a `number` were checked at every place ethers returns an integer. The chain id comparisons are all wrapped in `Number(...)`. `receipt.status` stayed a plain `number` in v6 (typed `number | null`), so `receipt.status !== 1` in `deployment-verification.service.ts` line 125 and `gm-verification.service.ts` line 216 is correct. `block.timestamp` stayed a plain `number`, so `block.timestamp * 1000` (deployment verification line 166, GM line 325) does not hit "Cannot mix BigInt and other types". `receipt.blockNumber` and `log.blockNumber` are plain numbers too, which is why `allLogs.sort((a, b) => b.blockNumber - a.blockNumber)` in `recent-domain.service.ts` line 337 still works.

String comparisons of decoded event values were checked in the v1 marketplace decoder. `event.args.listingId.toString()` gives the same decimal string from a `bigint` as it did from a `BigNumber`, so the comparisons against stored strings in `verifyNewSaleEvent` and `verifyListingAddedEvent` behave exactly as before. Marketplace v2 uses the same `.toString()` pattern for Seaport amounts (`seaport-event-application.service.ts` lines 341 to 354).

JSON serialisation of `bigint` was checked by searching for `JSON.stringify` of anything that could carry ethers values. None of the older modules return or log a raw `bigint`: gas estimates are converted with `Number`, USD prices go through `parseFloat(ethers.formatUnits(...))`, `parseUnits` results are only passed into `encodeDeploy`, decoded addresses and names are strings, and `configSnapshot.supply` stays the original string. Marketplace v2 handles `bigint` database columns with a `bigintStringTransformer` in `src/components/marketplacev2/order/entity/order.entity.ts` lines 6 to 10, covered in [03-seaport-primer-and-the-order-entity.md](03-seaport-primer-and-the-order-entity.md). There is no global `BigInt.prototype.toJSON` patch in the app, so any future code that returns a raw `bigint` from a controller will fail loudly with a 500, which is the safer failure mode.

## Behavioural risks the migration introduced or left open

These are the places where the v6 code compiles and passes its tests but can still misbehave in production. They are described in more depth in the per module notes linked from each paragraph.

The provider retry loop is the most widespread one. A v6 `JsonRpcProvider` constructed without a network hint discovers the chain lazily, and when its node is unreachable it logs "JsonRpcProvider failed to detect network and cannot start up; retry in 1s" and keeps retrying in the background. Marketplace v2 knows this, its comment at `src/components/marketplacev2/order/order.service.ts` lines 154 to 158 says "without it, a down/unreachable RPC makes the provider retry network detection every second forever", and it constructs one long lived provider with `{ staticNetwork: true }`. None of the older call sites do: `deployment-verification.service.ts` line 71, `contract-deployment.service.ts` line 278, `nft-collection.service.ts` line 313 and `gm-verification.service.ts` line 178 create a fresh provider on every request, and `recent-domain.service.ts` lines 53 to 57 create four long lived ones. The concrete failure is a misconfigured RPC secret turning a fast, clean "Failed to fetch transaction from chain" into a stalled request and a log line every second, and for deployment verification it can defeat the RPC failover loop. See [../05-blockchain-infrastructure/05-deployment-verification-the-shared-trust-boundary.md](../05-blockchain-infrastructure/05-deployment-verification-the-shared-trust-boundary.md).

`getBlock` returning `null` is the second. v6 types it as `Promise<Block | null>`, but this repository's `tsconfig.json` has no `strict` or `strictNullChecks`, so `let block: ethers.Block = await provider.getBlock(...)` compiles. A `null` there (a lagging node behind a load balancer) makes `block.timestamp` throw outside the surrounding `try` in both verifiers, turning a polite rejection into a 500. See [../08-affiliate-and-loyalty/04-gm-daily-checkin-and-streaks.md](../08-affiliate-and-loyalty/04-gm-daily-checkin-and-streaks.md).

`queryFilter` returning `EventLog | Log` is the third. In `recent-domain.service.ts` line 422, `log.args.uri` assumes every result decoded, and a plain `Log` has no `args`. See [../06-chain-and-registrar-integrations/05-not-actually-integrations.md](../06-chain-and-registrar-integrations/05-not-actually-integrations.md).

The ENS name trap is the fourth. The recent domains spec discovered that a v6 `Contract` given a target string that is not a hex address treats it as an ENS name and calls `provider.resolveName` on it straight away. `recent-domain.service.ts` never validates its four contract address config values, so a typo in Secrets Manager no longer simply fails, it becomes a background name lookup.

`getAddress` error handling is the fifth, older than v6 but carried through it. `gm.service.ts` line 330 normalises the wallet address before validating it and outside any `try`, so a malformed or badly checksummed address becomes a 500 rather than a 400. The deployment and renewal services already wrap `getAddress` in `try`/`catch` and throw `BadRequestException`, which is the pattern to copy.

## The arrival of a real test suite

The repository's own `CLAUDE.md` still says "No `.spec.ts` files found in the codebase as of this audit", which was out of date even at the baseline commit and is very out of date now. Counting with `git ls-files '*.spec.ts' | wc -l`, there were 86 spec files at `a131b429` and there are 148 on `uat`, an increase of 62. Fifty of the new ones live in `src/components/marketplacev2`. The other twelve were added around older code, and five older spec files were modified. Counting individual `it(...)` and `test(...)` cases, the suite grew from 594 to 1280, of which 616 are in marketplace v2 and 664 everywhere else. The single `test/app.e2e-spec.ts` end to end file is unchanged.

The specs added or changed outside marketplace v2 in this pull, and what each one covers:

| Spec file | Tests | Status | What it proves |
|---|---|---|---|
| `alchmey/arbAlchmey/arb-alchmey.servers.spec.ts` | 1 | new | `getMetaData` passes a v5 `BigNumber`, not a `bigint` or string, to `alchemy.nft.getNftMetadata` |
| `alchmey/fetch-expiry/keccak256-cross-agreement.spec.ts` | 1 | new | the two private `keccak256` helpers in fetch expiry and UD integration return the same hash |
| `alchmey/freename-alchmey/freename-alchmey.serveice.spec.ts` | 2 | changed | the Freename listing call now passes `timeout: EXTENDED_HTTP_TIMEOUT_MS` |
| `blockchain-deployment/services/deployment-verification.service.spec.ts` | 17 | new | every rejection path of all seven verification steps, the happy path, RPC failover, and `block.timestamp` arithmetic |
| `bnb-arb-domain-renewal/services/bnb-arb-domain-renewal.service.spec.ts` | 2 | new | golden calldata for `renew(string,uint256)` on BNB, and that different labels differ |
| `contract-deployment/services/contract-deployment.service.spec.ts` | 4 | new | `bigint` gas estimate becomes a `number`, fallback on RPC failure, bad wallet rejected before any RPC, golden `preparedCalldataHash` |
| `ens-integration/ens-integration.service.spec.ts` | 6 | changed | adds two tests that `Wallet.createRandom().address` is a valid checksummed address and is random |
| `eth-domain-renewal/utils/label-hash.util.spec.ts` | 2 | new | golden `labelHash('example')`, and different labels differ |
| `marketplace/abi/listing-id.util.spec.ts` | 5 | changed | adds a test that a listing id above `Number.MAX_SAFE_INTEGER` is accepted |
| `marketplace/abi/marketplace-event-decoder.spec.ts` | 15 | changed | fixtures rewritten with `bigint` literals and v6 helpers, Sprint 04 cases U4.1 to U4.7 |
| `nft-collection/services/nft-collection.service.spec.ts` | 4 | new | same four checks as the ERC20 spec, for the NFT collection prepare |
| `recentdomains/recent-domain.service.spec.ts` | 7 | changed | adds `bigint` chain id, `getChainName`, `.target` and `parseLogWithTimestamp` regressions |
| `reputation-gm-perk/gm/services/gm-verification.service.spec.ts` | 14 | new | every GM verification rejection reason and the happy path, with really encoded event logs |
| `reputation-gm-perk/gm/services/gm.service.spec.ts` | 4 | new | `getAddress` checksum normalisation, streak extend, streak reset, and month boundary rollover |

Three more new spec files outside marketplace v2, `src/@core/common/guards/require-verified-wallet.guard.spec.ts`, `src/components/auth/auth.service.spec.ts` and `src/components/web3-auth/web3-auth.service.spec.ts`, belong to the auth and identity notes in cluster 01. The marketplace v2 specs are spread across `market-data` (22 files), `order` (15), `poller` (6), `config` (2), `listing-status` (2), `transaction` (1), `wallet-verification` (1), and a module level `health-routes.guards.spec.ts`.

## How the suite is configured and how to run it

Jest is configured inside `package.json` itself, in the `"jest"` block at lines 143 to 168, not in a separate `jest.config.js`. The important settings are `"preset": "ts-jest"` and a `transform` entry so TypeScript is compiled on the fly, `"rootDir": "src"` so Jest only looks inside `src`, `"testRegex": ".*\\.spec\\.ts$"` so any file ending in `.spec.ts` is a test, `"testEnvironment": "node"`, and a `moduleNameMapper` that teaches Jest the same path aliases `tsconfig.json` defines, so that `import ... from 'src/...'` and `import ... from '@components/...'` resolve inside tests exactly as they do in the app. The tooling is `jest` `^29.7.0`, `ts-jest` `^29.4.11`, `@types/jest` `^29.5.14` and `@nestjs/testing` `^10.4.22`, which is newer than the Jest 27 the repository's `CLAUDE.md` still describes.

The npm scripts at `package.json` lines 17 to 21 are the way in. `npm run test` runs the whole suite once. `npm run test:watch` reruns affected tests as you save. `npm run test:cov` writes a coverage report to `../coverage` (one directory above `src`, so the repository root's `coverage` folder). `npm run test:debug` starts Jest under the Node inspector with `--runInBand` so you can attach a debugger. To run a single file or a group, pass a pattern through, for example `npm run test -- label-hash` or `npx jest src/components/reputation-gm-perk`. Because nearly every spec builds its service by hand with mocks, none of them need Postgres, AWS Secrets Manager, or a blockchain node, so the suite runs on a laptop with no secrets at all.

`npm run test:e2e` is a different story. It uses `test/jest-e2e.json`, which has no `moduleNameMapper`, so the `@components/...` imports inside `AppModule` cannot resolve, and even if they could, `test/app.e2e-spec.ts` is the untouched NestJS starter test expecting `GET /` to return "Hello World!", and booting `AppModule` needs real secrets and a database. Treat it as not working. Note also that nothing runs the suite automatically: `buildspec-backend.yml` only runs `npm install` and `npm run build`, and the husky `pre-commit` hook runs branch name validation, Prettier and ESLint, but not Jest. A failing test today blocks nobody unless a developer runs it by hand.

## The patterns the new specs use

Reading the new files side by side, a clear house style emerges, and it is worth learning because it is how you would write the next one.

Services are constructed by hand, not through NestJS's testing module. Only 13 of the 148 spec files call `Test.createTestingModule`, mostly in the analytics and search console modules. The rest do what `deployment-verification.service.spec.ts` lines 48 to 64 do: build a plain object for each dependency, with `jest.fn()` for each method the test needs, and pass them straight into the constructor with `as any`. That is fast, has no dependency injection magic to debug, and makes every dependency visible in the test file.

`ConfigService` is faked with a dictionary. The pattern is `const env = { BLOCKCHAIN_NETWORK: 'Mainnet', ... }; const configService = { get: jest.fn((key) => env[key]) } as any;`, seen in `arb-alchmey.servers.spec.ts`, `bnb-arb-domain-renewal.service.spec.ts`, `recent-domain.service.spec.ts` and `ens-integration.service.spec.ts`. `SecretsService` is faked the same way with `getSecret: jest.fn().mockResolvedValue({ POL_DEPLOY_RPC: '...' })`.

Network libraries are mocked at the module level, keeping everything else real. The ethers mock appears in the deployment, contract, NFT and GM specs:

```ts
// src/components/reputation-gm-perk/gm/services/gm-verification.service.spec.ts (lines 4 to 13)
jest.mock('ethers', () => {
    const actual = jest.requireActual('ethers');
    return {
        ...actual,
        ethers: {
            ...actual.ethers,
            JsonRpcProvider: jest.fn(),
        },
    };
});
```

Only `JsonRpcProvider` is replaced. `getAddress`, `keccak256`, `Interface` and the rest stay real, so the tests still exercise genuine v6 behaviour. Each test then calls `(ethers.JsonRpcProvider as unknown as jest.Mock).mockImplementation(() => makeProvider(...))` to decide what the fake node returns. The HTTP equivalent, used by the ENS, Freename, recent domains and fetch expiry specs, is `jest.mock('src/@core/utils/http-client.util', () => ({ httpClient: { get: jest.fn(), ... } }))`.

Private methods are tested through `as any`. `(service as any).estimateGas(...)`, `(service as any).encodeRenewCalldata(...)` and `(service as any).keccak256('test')` reach into private methods directly. Purists dislike this, but for a migration it is pragmatic, those private methods are exactly where the v5 calls lived.

Property injected dependencies are assigned by hand. `GmService` uses `@Inject(...)` on class properties rather than constructor parameters, so `gm.service.spec.ts` lines 33 to 36 assign `(service as any).gmCheckinRepository = ...` after construction.

Golden values pin outputs. `label-hash.util.spec.ts`, `bnb-arb-domain-renewal.service.spec.ts`, `contract-deployment.service.spec.ts` and `nft-collection.service.spec.ts` compare against fixed hex strings. Read the comments carefully though: several test titles say "matching the pre migration v5 output", while the comments underneath say the values were "computed once under v6 and pinned here". So they guarantee no future drift, but on their own they do not prove v5 and v6 agree; that rests on keccak256 and ABI encoding being unchanged algorithms, which they are.

Real encoding beats mocked decoding. `gm-verification.service.spec.ts` lines 21 to 32 build GM logs with `iface.encodeEventLog(fragment, values)`, and `marketplace-event-decoder.spec.ts` does the same for `NewSale` and `ListingAdded`, so the real `parseLog` runs on real bytes. Compare `deployment-verification.service.spec.ts`, which overwrites `ethers.keccak256` with a mock in `beforeEach` at line 67; that makes the calldata step easy to drive but means the real hash is never computed in that file.

Time is frozen where dates matter. Five spec files use `jest.useFakeTimers()` with `jest.setSystemTime(...)`, including `gm.service.spec.ts`, which proves a streak rolls from 28 February to 1 March 2026 correctly.

Marketplace v2 adds two heavier styles. `src/components/marketplacev2/poller/poller.integration.spec.ts` wires the real poller and both real application services together and fakes only Postgres (with a small stateful in memory store so one tick's writes are visible to the next) and the Polygon RPC (with a controllable fake provider). `src/components/marketplacev2/order/order.parity.spec.ts` runs all twelve order rejection cases plus the happy path end to end through `create()`, and is honest in its header about what it cannot do, fill a real order on mainnet. Both are covered in [08-the-onchain-event-poller-architecture.md](08-the-onchain-event-poller-architecture.md) and [03-seaport-primer-and-the-order-entity.md](03-seaport-primer-and-the-order-entity.md).

## Test ID conventions

Test names often begin with an ID tying them back to a written test plan. The most common family is `TC` followed by a section and case number, for example `it('TC2.1 every method sends x-api-key, the right base URL and the right path', ...)` at `src/components/marketplacev2/market-data/opensea/opensea.client.spec.ts` line 53, with IDs running from `TC2.1` up through `TC8.10` across the market data specs, and also used in older files such as `src/components/marketplace/payment/payment.service.spec.ts` and `src/components/freename/auth/freename-refreshtoken-repo.service.spec.ts`. A second family, `U` followed by numbers, appears in the v1 marketplace specs, `U4.1` to `U4.7` in `marketplace-event-decoder.spec.ts` and `U4.6` in `listing-id.util.spec.ts`, where the doc comments say they come from "Sprint 04". Marketplace v2's header comments also refer to work packages as `B-03` (orders) and `B-04` (the poller) and to numbered sprints inside each, and the ethers migration guards cite "Sprint 09" and "Sprint 11". When you add a test for a planned case, follow the same convention so the test plan and the code can be cross checked.

## What is still untested

It is worth being blunt about the gaps, because a 1280 test suite can give a false sense of safety. Thirty seven of the seventy eight folders under `src/components` have no spec file at all: `admin`, `ads`, `affiliate-user`, `analytics`, `aws-secrete`, `aws`, `bnb-integration`, `cancelled-order-diagnostics`, `cart`, `content-managment`, `custom-domain`, `email-verification`, `event-subscribe`, `event`, `events`, `fetch-domains`, `google-auth`, `healthCheck`, `invoice`, `landing_page`, `listener`, `localization`, `mail-send-meta-data`, `moralis`, `order-management`, `payment-management`, `provider-management`, `refund`, `search-management`, `subscribe`, `third-party-api`, `tlds-content-managment`, `twitter-auth`, `ud-billing`, `user`, `waitlist` and `wallet-address`. Several of those move money or identity (`cart`, `invoice`, `refund`, `payment-management`, `user`, `wallet-address`, `google-auth`). The `domain` folder, which holds the whole primary order flow, has a single spec file.

Beyond missing files, there are missing kinds of test. No spec talks to a real Postgres, so no TypeORM query, migration, unique constraint or transaction is ever exercised, including the delete then insert in `bulkUpdateDomainDetailBlockChain` and the new `findByTokenId` query builder. No spec boots `AppModule`, so wiring problems like the commented out `CronModule` described in [../10-ai-and-infra-utilities/05-cron-scheduled-jobs.md](../10-ai-and-infra-utilities/05-cron-scheduled-jobs.md) are invisible to the suite. No spec uses a real or local chain (the one Ganache based check, `src/scripts/verify-nft-template.js`, is a manual script outside Jest), so the v6 provider behaviours above, the retry loop, `getBlock` returning `null`, `queryFilter` returning plain logs, are not covered. Specific new code with no direct test includes `src/@core/utils/retry/is-retryable-chain-error.util.ts`, `BnbAlchmeyService.getOnChainExpiry` and its `"0"` edge case, the `findByTokenId` change, and the `CustomLoggerService.warn` console change. And because neither CodeBuild nor the git hooks run Jest, the suite only protects the code when someone remembers to run it, so a sensible next step is adding `npm run test` to `buildspec-backend.yml` before `npm run build`.

## Where to go next

For the module by module detail behind each row of the translation table, follow the update sections in [../05-blockchain-infrastructure/02-web3js-and-ethersjs-two-libraries-one-job.md](../05-blockchain-infrastructure/02-web3js-and-ethersjs-two-libraries-one-job.md), [../05-blockchain-infrastructure/04-contract-deployment-and-nft-collection-prepare-confirm-pattern.md](../05-blockchain-infrastructure/04-contract-deployment-and-nft-collection-prepare-confirm-pattern.md), [../06-chain-and-registrar-integrations/04-onchain-write-and-renewal-flows.md](../06-chain-and-registrar-integrations/04-onchain-write-and-renewal-flows.md) and [../03-commerce-and-marketplace/03-domain-listing-and-buy-domain-secondary-marketplace.md](../03-commerce-and-marketplace/03-domain-listing-and-buy-domain-secondary-marketplace.md). For how marketplace v2 uses v6 natively, start at [01-what-marketplace-v2-is-and-the-module-map.md](01-what-marketplace-v2-is-and-the-module-map.md).
