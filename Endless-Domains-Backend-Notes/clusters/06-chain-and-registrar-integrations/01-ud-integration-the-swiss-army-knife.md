# 01. ud-integration: the Swiss army knife

## What the name promises versus what the folder actually does

`ud-integration` sounds like it should be one thing: the bridge to Unstoppable Domains, the other Web3 domain registrar this company resells alongside its own product. For most of the folder that is exactly right. But at forty four files, this is the largest single folder in the entire cluster, nearly three times the size of the next biggest chain folder, and reading every file in it end to end turns up something the name does not warn you about: `ud-integration` has quietly become the one place in the codebase where several completely unrelated chains get their domain ownership and expiry looked up, using nothing but raw fetch calls the folder builds itself, with no shared abstraction connecting it back to the dedicated `aptos-integration`, `avax-integration`, `tezos-integration`, `starknet-integration`, or `ton-intigration` folders that supposedly already own that job.

## The part that is genuinely about Unstoppable Domains

`UDIntegrationService`, the 1400 line file at the center of the folder, constructs its base URLs and its bearer token straight from `ConfigService` in the constructor:

```ts
this.UD_BASE_URL = this.configService.get<string>('UD_BASE_URL');
this.SOL_BASE_URL = this.configService.get<string>('SOL_URL');
this.UD_AUTHORIZATION_TOKEN = this.configService.get<string>('UD_AUTHORIZATION_TOKEN');
```

Every real call to Unstoppable Domains follows the same shape after that: build a small enum value from `UDDomainAPI`, concatenate it onto `UD_BASE_URL` through a private `generateUDFullAPI` helper, attach `Authorization: Bearer <token>` as a header, and send it through the shared `httpClient` wrapper. `checkDomainAvailability` is the cleanest example of this:

```ts
async checkDomainAvailability(domainName: string, tlds: string): Promise<UDDomainCheckResponseDto> {
    const api = UDDomainAPI.DOMAIN_AVAILABILITY + domainName + '.' + tlds;
    const url = this.generateUDFullAPI(api);
    const config = { headers: { Authorization: 'Bearer ' + this.UD_AUTHORIZATION_TOKEN } };
    const response = await httpClient.get(url, config);
    return new UDIntegrationMutateData().toUDDomainCheckResponseDto(response.data);
}
```

That last line, handing the raw response to a fresh `new UDIntegrationMutateData()` instance every time, is a small but consistent habit across the whole file. `UDIntegrationMutateData` is a plain class with no injected dependencies and no state, so instantiating it per call costs nothing, but it does mean this mapping logic is never unit tested through dependency injection, only indirectly through the service's own spec file. Its job is entirely translation: `toUDDomainCheckResponseDto` reads Unstoppable Domains' own `availability.status` enum (`AVAILABLE`, `REGISTERED`, `RESERVED`, `PROTECTED`, `DISALLOWED`, `COMING_SOON`) and maps it onto the company's own `UDDomainCheckResponseDto`, filling in `ownerAddress` only when the domain is actually registered.

The order flow (`orderDomain`, `orderDomainStatus`) and the newer v3 claim and transfer flow (`claimDomain`, `domainTransfer`, `returnAndRefundDomain`) all follow that same request, then mutate, then return shape, just with `POST`, `PUT`, and `DELETE` in place of `GET`. The v3 methods add one more layer worth noticing: they fan a list of domains out into parallel private helper calls (`callUDV3ClaimDomain`, `callUDV3DomainTransfer`, `callUDV3ReturnAndRefundDomain`) with `Promise.all`, and each of those helpers deliberately never rejects, it catches its own error and returns a plain object with `domainOrderStatus: false` instead, so one failed domain in a bulk claim never breaks the other nine.

Two operational touches are worth calling out because they show real production experience rather than a tutorial pattern. First, `getDomainNamesInEndlessDomainWallet` and its two admin variants all have to walk Unstoppable Domains' own cursor pagination by hand:

```ts
let tempCursor = response?.data?.next?.['$cursor'] || null;
while (tempCursor) {
    const innerResponse = await httpClient.get(url, { params: { $cursor: tempCursor }, headers: ... });
    domainName.push(...(innerResponse?.data?.items || []));
    tempCursor = innerResponse?.data?.next?.['$cursor'] || null;
}
```

The admin variant even adds a `visitedCursor` set specifically to guard against an infinite loop if the API ever hands back the same cursor twice, a defensive touch the other two copies of this same loop do not have, which is itself a small sign that this logic has been patched more than once without ever being pulled into one shared helper. Second, `reserveDomainsFromS3Excel` is a genuinely unusual feature for an "integration" folder to own: it downloads an Excel file of domain names from S3, calls the UD reserve endpoint once per row, and uploads a second Excel file of successes and failures back to S3, entirely so someone on the business side can bulk reserve a list of names on Unstoppable Domains without writing any code themselves.

## The part that has nothing to do with Unstoppable Domains

This is the deviation worth understanding, not glossing over. Buried in the same service, `getDomainExpiry` is a router that decides, based on a `domainProvider` string parameter, which of seven completely different lookup strategies to run:

```ts
if (domainProvider === 'UD') {
    const owner = await this.checkOwnerOfUdDomain(domainName);
    ...
} else if (domainProvider === 'Bonfida') {
    const owner = await this.getSolDomainOwner(domainName);
    ...
} else if (domainProvider === 'Tezos') {
    const response = await this.getTEZDominOwner(domainName);
    ...
} else if (domainProvider === 'Aptos') {
    ...
} else if (domainProvider === 'Ton') {
    ...
} else if (domainProvider === 'Avax') {
    ...
} else if (domainProvider === 'Starknet') {
    ...
} else {
    // .eth, .bnb, .arb: read the base registrar contract directly with web3
}
```

Every one of those branches calls a private method that this same file also owns, and every one of those private methods reimplements a call this codebase already has a dedicated folder for. `getTEZDominOwner` posts the exact same GraphQL query as `tezos-integration`, using its own copy of the query string stored at `queries/domain-detail.query.ts`. `getAptosDominOwner` and `getAvaxDominOwner` do the same thing against copies of the Aptos and Avax GraphQL queries, stored again at `queries/aptos-details-query.ts` and `queries/avax-details-query.ts`, byte for byte identical to the query constants living inside `aptos-integration/queries/aptos-queries.ts` and `avax-integration/queries/avax-queries.ts`. `getTonDominOwner` and `fetchTheOwnerOfStarkentDomain` build their own URLs against the Ton and Starknet endpoints rather than calling into `ton-intigration` or `starknet-integration` at all. Even the fallback branch, for `.eth`, `.bnb`, and `.arb`, instantiates its own `Web3` clients against three RPC providers and reads the base registrar contract directly, using ABI JSON files this folder also happens to keep a copy of, at `ud-integration/contracts/ENSBaseRegistrarContract.json`, `BNBBaseRegistrarContract.json`, and `ARBBaseRegistrarContract.json`.

The likely reason this exists is visible in the constructor and the DTO shapes: `UDIntegrationService` is the one service the rest of the domain product actually calls to answer "who owns this domain and when does it expire" regardless of which chain or registrar the domain lives on, so at some point it was easier to fold every chain's expiry lookup into this one already central file than to add a dependency on eight other modules. The cost of that choice is real: a bug fix to the Tezos GraphQL query has to be made in two places, `tezos-integration` and here, and nothing in the code enforces that anyone remembers to do both. This is exactly the kind of thing worth flagging to a reviewer rather than assuming is intentional, because it reads much more like organic growth than a deliberate design.

## The rest of the folder, quickly

The `dto` folder (twenty three files) is almost entirely flat data classes with no logic, one per shape Unstoppable Domains' API or this service's own callers need, plus a handful of validated request DTOs (`ReserveDomainDto`, `resercveAutomationDto`, `DomainListQueryDto`) that use `class-validator` decorators the way the rest of the codebase does. `UDDomainAPI`, the enum holding every Unstoppable Domains endpoint path, has two pairs of intentionally duplicated string values (`DOMAIN_AVAILABILITY` and `UD_DOMAIN_OWNER` both point at `/domains/`, and `RESERVE_DOMAIN_ON_UD` and `RELEASE_RESERVE_DOMAIN_ON_UD` both point at `domains`), each one commented with an eslint disable explaining why the duplication is fine, since the actual HTTP method and any trailing path segment is what tells the two calls apart. The controller is a thin pass through, one route per public service method, guarded by `AccessTokenGuard` and occasionally `AdminTokenGuard`, wrapping every response in the shared `Response` DTO the rest of the API uses. Nothing in the controller or module is unusual, they are exactly the shape you would expect from any other NestJS feature module in this codebase.
