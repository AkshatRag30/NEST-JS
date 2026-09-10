# 03. The shared checkAvailability pattern, and its one broken copy

## Reading one of these all the way through

Of the ten small folders in this cluster, `starknet-integration` is the cleanest example of the shape every other one is built from, so it is worth reading in full once rather than skimming ten near identical services. The whole module is four files: an interface, a DTO, a module, and the service itself. The service reads one config value in its constructor, `STARKNET_API_URL`, and exposes exactly one public method:

```ts
public async checkAvailability(domainName: string): Promise<StarknetResponseDTO | null> {
    try {
        const response = await httpClient.get(`${this.STARKNET_API_URL}?domain=${domainName}.stark`, {
            timeout: 4000,
        });
        return response.data;
    } catch (error) {
        const errorMsg = error.code === 'ECONNABORTED'
            ? `Timeout: Starknet API took too long to respond for domain ${domainName}`
            : `Error fetching Starknet availability: ${error.message}`;
        this.customLoggerService.log(errorMsg);
        logger.error(errorMsg);
        return null;
    }
}
```

That is the entire pattern. A base URL from config. A suffix appended for the chain's own TLD (`.stark` here). One outbound call through the shared `httpClient` wrapper. A catch block that never lets the error escape the service, choosing instead to log it and hand the caller back a value that reads as "could not verify, treat as available" rather than a thrown exception, a choice the comment directly above the return spells out on purpose. Everything else in the file, the interface and the DTO, exists purely to give that one method a typed signature.

## What actually varies from one folder to the next

Nine other folders repeat this shape with three things changing: which HTTP verb and payload style the target API expects, what the response looks like on the way back, and how defensive the error handling turns out to be. The table below is built from actually reading every service file, not from assuming the folder names imply sameness.

| Folder | What it calls | Request style | On success | On failure |
|---|---|---|---|---|
| `starknet-integration` | Starknet resolver API | GET with a 4 second timeout | returns `response.data` untouched | logs, returns `null` |
| `box-integration` | Box Domains REST API | GET with an `x-api-key` header | remaps fields into `BoxResponseDTO` (`available`, `name`, `is_premium`, `href`, `status`) | logs, then rethrows the original error |
| `bonfida` | QuickNode Solana RPC | JSON RPC POST, `sns_resolveDomain`, with three retries and a 500ms backoff | returns `response.data.result` (a wallet address) | after three attempts, logs and returns the literal string `'error'` |
| `avax-integration` | Avvy Domains GraphQL API | GraphQL POST, `GetDomainInfo` | returns `response.data.data` as is | logs, returns `{ domains: [] }` |
| `aptos-integration` | Aptos Names GraphQL API | GraphQL POST, `domain_availability_search` | returns the first row of `current_ans_lookup_v2` | logs, returns `undefined` |
| `ton-intigration` | TON DNS resolver | GET, no config object at all | returns `response.data` untouched | logs, returns `undefined` |
| `tezos-integration` | Tezos Domains GraphQL API | GraphQL POST with a hand built query string | returns `response.data.data.domain` | logs, returns `undefined` |
| `ens-arb-bnb-domain-suggestion` | a shared NFT marketplace GraphQL API, used for ENS, Arbitrum, and BNB domains together | GraphQL POST, `domains` query with `exactMatch`/`list`/`pageInfo` | returns `response.data.data.domains` | logs, returns `[]` |
| `arb-integration` | the same shared marketplace GraphQL API as above | GraphQL POST, identical query | never returns anything to its caller, see below | logs the error object, still returns nothing |
| `bnb-integration` | nothing at all | n/a | a single `domainTransfer()` method that logs a start message and resolves | n/a |

A few things are worth pulling out of that table rather than leaving buried in it. Every single one of these services builds its own request by hand: nobody shares a base "call an external API and map the response" helper across folders, each service writes its own `httpClient.get` or `httpClient.post` call with its own header and body shape, even when three of them (`avax`, `aptos`, `tezos`) are hitting near identical GraphQL shaped endpoints. And error handling is genuinely inconsistent in a way that matters: some services swallow every error into a safe fallback value (`starknet`, `avax`, `ton`, `tezos`, `aptos`, `ens-arb-bnb-domain-suggestion`), while `box-integration` is the one exception that logs and then rethrows, meaning a Box API outage will actually surface as a 500 to whatever controller called it, unlike every other chain in this table degrading quietly to "treat as unavailable to verify."

## The one real deviation: a duplicated, broken suggestion call

`arb-integration` and `ens-arb-bnb-domain-suggestion` are not two different integrations, they are the same integration written twice. Both files build the exact same GraphQL query, word for word, against a marketplace API configured through the same `ARB_BRB_GRAPHQL_API` environment variable, asking for `exactMatch`, `list`, and `pageInfo` on a set of ARB and BNB domain listings. The version inside `ens-arb-bnb-domain-suggestion` is the one that actually works as a usable method:

```ts
const response = await httpClient.request(config);
return response.data.data.domains;
```

It awaits the request and returns the parsed result to whatever called `getEnsArbBnbSuggestion`. The version inside `arb-integration`, `getArbAndBnbSuggestion`, sends the identical request but never awaits it and never returns anything:

```ts
httpClient
    .request(config)
    .then((response) => { logger.log(response.data); })
    .catch((error) => { logger.log(error); });
}
```

The method's own return type is declared `any`, but at runtime it returns `undefined` synchronously every time, because the whole body is a fire and forget promise chain that only ever logs its own result, it never hands anything back to a controller or another service. Anyone calling `ArbIntegrationService.getArbAndBnbSuggestion` today gets nothing useful out of it no matter what the API actually returns. Given that `ens-arb-bnb-domain-suggestion` exists as a separate, working module doing the identical job, the strong likelihood is that `arb-integration`'s copy is an earlier, abandoned attempt at the same feature that nobody deleted once the real version was built next to it. Its sibling method on the same service, `domainTransfer()`, is even more clearly a placeholder, a single line that logs `'arb integration start '` and returns, with no body at all.

`bnb-integration` is the extreme version of the same story: the entire service is one method, `domainTransfer()`, that does nothing but instantiate a `Logger` and log a start message. There is no controller, no DTO, and nothing calling it anywhere else in the codebase that turned up in this read. Between `arb-integration` and `bnb-integration`, this cluster is carrying two module scaffolds that exist structurally, register correctly in Nest's dependency injection container, and do genuinely nothing, which is worth knowing before assuming every folder with an "integration" suffix in its name is load bearing.

## ens-integration and ens-arb-bnb-domain-suggestion are not the same thing

One naming trap worth flagging on its own: `ens-integration` and `ens-arb-bnb-domain-suggestion` sound like variations on the same feature, but they do not overlap at all. `ens-arb-bnb-domain-suggestion` fits everything in this file, a small, single method GraphQL availability and suggestion lookup. `ens-integration` does not fit this pattern in any way, it is a full commit and reveal domain registration flow that signs and sends real transactions with a company held wallet, and it is covered on its own in the next note alongside the two renewal folders, because it belongs with them, not with this table.
