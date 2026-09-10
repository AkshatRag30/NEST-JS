# 07. Moralis and Cross Chain NFT Lookups

## The same kind of company as Alchemy, a second, older dependency

Moralis is, at the level that matters here, the same category of product as Alchemy, a third party blockchain data provider with its own indexed API for questions like "what NFTs does this wallet own on this chain," reached here through the `moralis` npm package (version `^2.27.2`) and its companion `@moralisweb3/common-evm-utils` package. Unlike `alchmey`, there is only one `moralis` folder, not one per chain, `moralis.service.ts` itself branches internally on which chain it has been asked about. `main.ts`, covered in the architecture notes for this project, initializes Moralis exactly once at application boot, `await Moralis.start({ apiKey: secrets.MORALIS_API_KEY })`, the same "connect once, not per request" pattern used for the database connection, rather than constructing a fresh client per request the way each Alchemy service does.

## One function, branching per chain

`callMorallisApi` is the heart of the service, and its `switch` statement is the clearest evidence of how many different chains a single Moralis integration has to account for:

```ts
switch (domainprovider) {
    case 'UD':
        chain = this.BLOCKCHAIN_NETWORK == 'Testnet' ? EvmChain.MUMBAI : EvmChain.POLYGON;
        break;
    case 'ENS':
        chain = this.BLOCKCHAIN_NETWORK == 'Testnet' ? EvmChain.GOERLI : EvmChain.ETHEREUM;
        break;
    case 'Arbitrum':
        chain = EvmChain.ARBITRUM;
        break;
    case 'BinanceSmartChain':
        chain = EvmChain.BSC;
        break;
    default:
        chain = EvmChain.GOERLI;
}
```

`EvmChain` is an enum from Moralis' own SDK identifying which chain's indexed data to query, and this switch is effectively a map from this company's own domain provider naming (`UD` for Unstoppable Domains, `ENS`, `Arbitrum`, `BinanceSmartChain`) onto Moralis' chain identifiers, including separate testnet variants for local development and staging. Worth noticing as an honest, small rough edge, the `BinanceSmartChain` testnet branch actually maps to `EvmChain.BSC` (a mainnet identifier) while labeling itself `'Goerli Testnet'` in a variable used only for logging, a mismatch that would only matter if someone actually ran this path against a testnet, and a good example of the kind of small inconsistency you find constantly in a real, several years old codebase rather than a cleaned up teaching example.

## Reading a wallet's NFTs, with manual pagination

```ts
const response = await Moralis.EvmApi.nft.getWalletNFTs({ address: walletaddress, chain, limit: 100 });
if (response.result.length == 0) {
    return [data, true];
} else {
    if (response.pagination.cursor == null) {
        data.push(...response.result);
    } else {
        let cursor = response.pagination.cursor;
        data.push(...response.result);
        while (cursor != null) {
            const innerResponse = await Moralis.EvmApi.nft.getWalletNFTs({ address, chain, limit: 20, cursor });
            ...
        }
    }
}
```

This is the same underlying idea as Alchemy's `pageKey` loop in file 06, Moralis calls its cursor `cursor` instead, but the shape is identical, ask for a page, keep asking for the next page using the cursor the previous response handed back, until a response comes back with no cursor left, meaning every NFT the wallet owns on that chain has now been collected into `data`.

## Turning a raw NFT list into this company's own shape

`filterMoralisData` is where the raw provider response gets narrowed down to only the tokens this company actually cares about, and enriched with information Moralis has no way to know:

```ts
if (smartContractsArray.includes(morallisObject.tokenAddress._value)) {
    ...
    switch (blockchain) {
        case DomainProvider.ENS:
            domainName = (await this.ensWeb3Service.getMetaData(tokenId)).name;
            break;
        case DomainProvider.Arbitrum:
            domainName = (await this.arbWeb3Service.getMetaData(tokenId)).name;
            break;
        case DomainProvider.BinanceSmartChain:
            domainName = await this.getBSCMetaData(tokenId);
            break;
        ...
    }
    let domainExpiry = await this.fetchExpiryService.getDomainExpiry(domainName);
    ...
}
```

Just like the Alchemy path, `smartContractsArray` (built from this company's own `domain-provider-smart-contract.repo.service.ts` table) filters out any NFT that is not one of this company's own registrar contracts, since a wallet could hold plenty of other, unrelated NFTs Moralis would happily also report. For each surviving NFT, `MoralisService` actually reaches back into this cluster's own per chain `web3` services (`EnsWeb3ServiceInterface`, `ARBWeb3ServiceInterface`, covered in file 08) to resolve a human readable domain name, and into `FetchExpiryServiceInterface` (from the `alchmey` folder, shared across both providers) to find the domain's expiry date. This is worth sitting with, Moralis answers "what tokens exist," but this company's own contracts and services are still the source of truth for "what domain name does this token actually represent, and when does it expire," a third party data provider narrows the search space, it does not replace this company's own on chain knowledge.

## A direct metadata call, and a Solana branch that is not EVM at all

```ts
async getBSCMetaData(tokenId: string): Promise<string> {
    const chain = EvmChain.BSC;
    const response = await Moralis.EvmApi.nft.getNFTMetadata({ address: /* the BSC registrar contract */, chain, tokenId });
    ...
}
```

is the single token equivalent seen already in file 06 for Alchemy, `getNFTMetadata` rather than `getWalletNFTs`, answering a question about one specific token rather than an entire wallet.

`nftsOwnedbySolanaWalletAddress` is worth calling out for a different reason, it does not use Moralis' EVM API at all, because Solana is not an EVM chain, it has an entirely different transaction and account model. That method instead delegates straight to `BonfidaIntegrationService` (a separate integration covered elsewhere in this project's notes), and the result gets reshaped through `createMoralisDetailData` into the same `moralisDetailResponseDto` shape the EVM path produces. This is a small but telling example of a wider pattern worth carrying forward, "Web3" is not one uniform technology, EVM chains (Ethereum, Arbitrum, BNB Smart Chain, Polygon, and most of the chains named throughout this cluster) all share a broadly similar model that libraries like `web3.js`, `ethers.js`, and Moralis' `EvmApi` can speak uniformly, but Solana, and several of the other chains this company integrates with elsewhere in the codebase, do not, and need an entirely separate client and mental model.

## Where Moralis and Alchemy actually differ in practice, in this codebase

Reading both folders side by side, the honest answer is that they overlap heavily in capability, both can list a wallet's NFTs and fetch a single token's metadata, and this codebase uses each for a different, specific set of chains rather than picking one globally. `MoralisModule`'s own constructor injects `EnsWeb3ServiceInterface`, `UDWeb3ServiceInterface`, `BNBWeb3ServiceInterface`, and `ARBWeb3ServiceInterface` together, suggesting Moralis is the path this company built first, and, per the comments seen throughout `alchmey`, Alchemy was layered in later, chain by chain, likely as the team learned where one provider's coverage, pricing, or reliability made more sense than the other for a specific chain. Neither file claims this outright, but it is the most coherent explanation the evidence in both folders supports, and it is a useful lesson in its own right, in a real production system, "why do we have two of the same kind of dependency" often has an answer rooted in when each integration was built and what tradeoffs made sense at that specific time, not a design flaw to immediately consolidate.

## Where to go next

[08-per-chain-web3-services-and-ipfs.md](08-per-chain-web3-services-and-ipfs.md) covers the smaller per chain services this file leaned on for domain name resolution, `EnsWeb3Service` and `ARBWeb3Service`, along with the rest of the `web3` folder, `web3-transaction` and `ipfs`.
