# 06. Quirks and bugs catalog

The rest of this cluster's notes each mention one or two of these in context, where they matter most. This file collects every one of them in a single place, specifically because they are the kind of thing that only surfaces from actually reading every file rather than trusting a folder's name or its size. None of these are hypothetical, each one is quoted or pointed at a real file.

## The dead, unreachable suggestion method

Covered in full in [03-the-shared-checkavailability-pattern.md](03-the-shared-checkavailability-pattern.md). `ArbIntegrationService.getArbAndBnbSuggestion`, inside `arb-integration/arb-integration.service.ts`, sends the same GraphQL request that `EnsArbBnbDomainSuggestionService.getEnsArbBnbSuggestion` (in the separate `ens-arb-bnb-domain-suggestion` folder) sends, but never awaits or returns the result, only logs it inside a floating `.then`/`.catch` chain. Anything calling this method today gets nothing back no matter what the API returns. This is the single most likely genuine bug in the whole cluster, because the working version sitting right next to it in a different folder makes it very clear what the broken one was supposed to do.

## Two module scaffolds that do nothing at all

`bnb-integration` is a complete NestJS module, wired correctly into dependency injection with an exported interface, whose only method logs a start message and returns. `arb-integration`'s second method, `domainTransfer()`, is the identical shape, a `Logger` call and nothing else. Neither has a controller route that a frontend could reach, and neither turned up any real caller elsewhere in this cluster during this read. They exist, compile, and register, and do nothing.

## A misspelled provider token: Bonfida becomes Bondifa

`bonfida/bonfida-integration.module.ts` provides and exports its service under the token `'BondifaIntegrationServiceInterface'`, and the module class itself is named `BondifaIntegrationModule`, both a letter swap away from the folder's own name and from every comment and log message inside the service, which all spell it correctly as Bonfida. Anywhere else in the codebase that needs to inject this service has to know to ask for the misspelled token, which is exactly the kind of inconsistency that costs someone fifteen confused minutes the first time they go looking for it.

## A copy pasted GraphQL query that a REST integration never runs

`box-integration/queries/box-queries.ts` defines `BOX_DOMAIN_DETAIL_QUERY`, a full GraphQL query and fragment for `current_ans_lookup_v2`, the exact query used by `aptos-integration` and echoed inside `ud-integration`'s own Aptos lookup path. `box-integration.service.ts` never imports or references this constant anywhere; the Box service only ever makes one plain REST `GET` call to a `/check-domain` endpoint. The query file was almost certainly copied over as a starting template from the Aptos folder and never replaced or deleted once the real Box integration turned out not to need GraphQL at all.

## Two DTO files that exist but hold nothing

`bonfida/dto/ApiResponse.dto.ts` and `ens-integration/ens-integration-mutate-data.service.ts` are both real, tracked files with a `.ts` extension and zero content, no class, no export, nothing. Neither one is imported anywhere. They read as leftovers from an earlier version of each folder's structure, most likely a mutate data helper file for `ens-integration` planned to mirror `ud-integration-mutate-data.ts`'s pattern, and a response DTO for Bonfida that never ended up getting written.

## Freename's duplicated lifecycle status enum

Covered in [02-freename-three-modules-one-registrar.md](02-freename-three-modules-one-registrar.md). `freename/domain/enum/freename-lifecycle-status.enum.ts` exports a `FreenameLifecycleStatus` enum with five values (`REQUESTED`, `PENDING`, `REGISTERED`, `FAILED`, `EXPIRED`). A second, completely different `FreenameLifecycleStatus` enum, with nine values covering the same states plus four minting specific ones, is declared inline inside `freename-domain-lifecycle.entity.ts`, and that second one is the one every real service in the folder actually imports and uses. The standalone enum file is dead, but it shares its exact type name with the live one, so an editor autocomplete or a careless import could easily pull in the wrong one.

## A live RPC URL with a project id hardcoded in source

Covered in [05-not-actually-integrations.md](05-not-actually-integrations.md). `recentdomains/recent-domain.service.ts` constructs its BNB Chain provider with a literal Infura URL, complete with what reads as a real project id, `https://bsc-mainnet.infura.io/v3/b1d4662319b343259d5e71620840485a`, instead of reading `BNB_URL` from config the way it does for every other chain in the same constructor. Whether or not that specific id is still active, committing any RPC provider key straight into source rather than through `ConfigService` is the kind of thing worth raising, since it bypasses whatever secrets rotation process the rest of the app's config values go through.

## A naming trap between two similarly named folders

Not a bug, but worth repeating here because it is easy to trip over: `ens-integration` and `ens-arb-bnb-domain-suggestion` are not variations on the same feature despite the overlapping name. One is a full custodial commit and reveal registration flow that signs real transactions with the company's own wallet; the other is a small, read only GraphQL availability and suggestion lookup shared across ENS, Arbitrum, and BNB. Assuming they are close cousins because of the shared `ens` prefix is exactly the kind of folder name assumption this whole cluster turned out to reward checking rather than trusting.

## Update from the October 2026 uat pull

Every entry above is still accurate after the October 2026 pull, including the hardcoded Infura key, which commit `71fd2fec` touched (it changed `ethers.providers.JsonRpcProvider` to `ethers.JsonRpcProvider` on that very line, `recent-domain.service.ts` line 54) without moving the key into config. The ethers v5 to v6 upgrade in commits `71fd2fec` and `26b1f0e4`, plus a few unrelated fixes, add the following entries to this catalog. Each one is explained in depth in the note named in brackets.

### ENS quote generation passed a meaningless argument to `createRandom`

`ens-integration/ens-integration.service.ts` line 227 used to call `ethers.Wallet.createRandom(['i'])`. The array was never a valid argument, v5 silently ignored it and v6 rejects it at compile time, so commit `26b1f0e4` removed it. This one is now fixed, and listed here so that anyone reading the old code in git history understands the change ([04-onchain-write-and-renewal-flows.md](04-onchain-write-and-renewal-flows.md)).

### A `bigint` chain id quietly broke the UD ticker, and was fixed

In ethers v6 `network.chainId` is a `bigint`, so the old `network.chainId === 137` in `recentdomains/recent-domain.service.ts` `testUdNetwork` would always have been `false`, because `137n === 137` is `false`. Line 153 now reads `Number(network.chainId) === 137`. Fixed, and pinned by two new tests in `recent-domain.service.spec.ts` ([05-not-actually-integrations.md](05-not-actually-integrations.md)).

### `getRecentBnbDomains` can return `undefined` and sink the whole homepage ticker

Still open. `recentdomains/recent-domain.service.ts` lines 246 to 313 retry three times and then fall off the end of the function without a `return`, so the caller gets `undefined`. `seedAllRecentDomains` only guards against thrown errors, so `bnbDomains.slice(0, 3)` at line 465 throws and the whole refresh of `tbl_recent_domain` fails even when the other four sources succeeded ([05-not-actually-integrations.md](05-not-actually-integrations.md)).

### UD log decoding assumes every log decoded

Still open. `queryFilter` in v6 returns `EventLog | Log`, and only `EventLog` has `.args`, so `log.args.uri` at `recent-domain.service.ts` line 422 throws on any log the ABI could not decode, failing the whole UD feed rather than skipping that log ([05-not-actually-integrations.md](05-not-actually-integrations.md)).

### Provider objects without `staticNetwork`

Still open, and new with v6. The older integrations in this cluster build `new ethers.JsonRpcProvider(url)` without a chain id or `{ staticNetwork: true }` (four providers in `recent-domain.service.ts` lines 53 to 57). In v6 an unreachable URL makes such a provider retry network detection once a second indefinitely, logging each time, which the newer marketplace v2 code explicitly guards against at `src/components/marketplacev2/order/order.service.ts` lines 154 to 159 ([../05-blockchain-infrastructure/02-web3js-and-ethersjs-two-libraries-one-job.md](../05-blockchain-infrastructure/02-web3js-and-ethersjs-two-libraries-one-job.md)).

### Two renewal verifiers compare hashes case sensitively across two libraries

Still open, latent rather than live. `eth-domain-renewal/services/eth-domain-renewal-verification.service.ts` lines 38 to 39 and `bnb-arb-domain-renewal/services/bnb-arb-domain-renewal-verification.service.ts` lines 41 to 42 compare a web3.js `keccak256` of the on chain input against an ethers v6 `keccak256` stored at prepare time using a plain `!==`. Both libraries emit lowercase hex today, so it works, but nothing tests that agreement ([04-onchain-write-and-renewal-flows.md](04-onchain-write-and-renewal-flows.md)).

### The BNB expiry lookup can overwrite a good value with `"0"`

Still open, new in commit `546e26e9`. `alchmey/bnb-alchmey/bnb-alchmey.serveice.ts` line 237 returns `result || null` from the registrar's `nameExpires`, and the string `"0"` is truthy, so a label the configured registrar does not know replaces the indexer's real expiry with 1 January 1970 ([../05-blockchain-infrastructure/06-alchemy-and-the-alchmey-folder.md](../05-blockchain-infrastructure/06-alchemy-and-the-alchmey-folder.md)).

### Undeclared v5 helper packages

Still open. `@ethersproject/bignumber` (imported at `alchmey/arbAlchmey/arb-alchmey.servers.ts` line 11) and `@ethersproject/bytes` (imported at `alchmey/fetch-expiry/fetch-expiry.service.ts` line 11 and `ud-integration/ud-integration.service.ts` line 27) are not in `package.json`. They used to arrive with `ethers@5`, and since the upgrade they only arrive because `alchemy-sdk` depends on them. The codebase wide picture is in [../12-marketplace-v2/12-ethers-v6-migration-and-the-new-test-suite.md](../12-marketplace-v2/12-ethers-v6-migration-and-the-new-test-suite.md).
