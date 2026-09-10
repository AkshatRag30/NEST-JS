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
