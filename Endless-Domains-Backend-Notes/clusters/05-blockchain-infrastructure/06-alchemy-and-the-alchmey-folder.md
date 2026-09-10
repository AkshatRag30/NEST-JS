# 06. Alchemy and the `alchmey` Folder

## Why a backend would ever need a company like Alchemy

Running your own blockchain node, a full copy of a chain's entire history, kept continuously in sync, is a genuinely heavy piece of infrastructure, expensive to run, and easy to get wrong. Most companies that need to read from or write to a blockchain do not run their own node at all, they pay a third party that already runs a large, reliable fleet of nodes, and exposes a normal HTTPS or SDK based API in front of them. Alchemy is one of the largest companies doing exactly this, "blockchain data provider" is the right way to think about it, a company that lets an ordinary backend ask questions like "what does this address own" or "send this transaction," without that backend ever running a node itself. This codebase's `package.json` lists `alchemy-sdk` at version `^3.5.0`, and the folder is named `alchmey` (a misspelling that persists throughout every file in it, worth knowing so it does not look like a typo you introduced).

## The shape every per chain Alchemy service shares

The `alchmey` folder holds one subfolder per chain this feature covers, `arbAlchmey`, `bnb-alchmey`, `ens-alchmey`, `freename-alchmey`, `ud-alchmey`, `ud-base-alchmey`, plus a shared `fetch-expiry` helper (covered again in file 08) and a `domain-provider-smart-contract.repo.service.ts` repeated in each folder, a small repository that looks up which on chain registrar contract address corresponds to which domain provider and network, read from this company's own database rather than hard coded. Every chain's service follows the same shape as `arb-alchmey.servers.ts` (again, spelled exactly that way in the codebase):

```ts
import { Alchemy, Network } from "alchemy-sdk";
...
this.config = {
    ALCHMEY_KEY: this.configService.get<string>('ALCHMEY_KEY'),
    network: Network.ARB_MAINNET,
};
this.alchemy = new Alchemy(this.config);
```

One `Alchemy` client instance is constructed per service, configured with an API key pulled from configuration (never hard coded) and a specific `Network` enum value telling Alchemy which chain's nodes to route requests to. From there, the service asks Alchemy's SDK for exactly the two things it needs.

## Reading every NFT a wallet owns

```ts
async fetchNFTs(ownerAddress: string, blockchain: string, userID: string): Promise<any> {
    const options = { contractAddresses: [/* this chain's domain registrar contract */] };
    const allNfts: any[] = [];
    let pageKey: string | undefined = undefined;
    do {
        const nfts: any = await this.alchemy.nft.getNftsForOwner(ownerAddress, { ...options, pageKey });
        allNfts.push(...nfts.ownedNfts);
        pageKey = nfts.pageKey;
    } while (pageKey);
    ...
}
```

`this.alchemy.nft.getNftsForOwner` is doing something that would be genuinely painful to do by hand against a raw node, scanning an entire chain's history for every NFT a given address currently holds. Alchemy indexes that ahead of time and just answers the query directly. `contractAddresses` narrows the request down to only this chain's own domain registrar contract, since this service only cares whether the wallet owns one of this company's own domain NFTs, not every NFT that wallet might hold anywhere. `pageKey` is a cursor, Alchemy hands results back a page at a time, and the `do...while` loop keeps asking for the next page until `pageKey` comes back empty, meaning there is nothing left to fetch.

Once the raw NFTs come back, `filterMoralisData` (a name left over from an earlier, shared implementation, despite living in the Alchemy service) cross references each NFT's contract address against this company's own `domain-provider-smart-contract.repo.service.ts` lookup, keeping only the ones that actually belong to a registrar this company recognizes, then calls out to a completely separate, third party GraphQL API (`space.id`) to fetch the domain's expiry date. That last detail matters, Alchemy tells you what a wallet owns, it does not know anything about this company's own domain expiry logic, that still has to come from elsewhere.

## Reading one NFT's metadata directly

```ts
async getMetaData(tokenId: string, contractAddress: string): Promise<any> {
    const bigNumberTokenId = BigNumber.from(tokenId);
    const metadata = await this.alchemy.nft.getNftMetadata(contractAddress, bigNumberTokenId);
    return metadata;
}
```

`getNftMetadata` is the single token equivalent of `getNftsForOwner`, given a contract address and a token ID, Alchemy resolves and returns that token's metadata (its `tokenURI` contents, already fetched and parsed on Alchemy's side) without this backend needing to call the contract itself or fetch and parse an IPFS or HTTP URI on its own.

## What this backend never asks Alchemy to do

Every method across every `alchmey` service folder is a read. Nothing in this folder ever asks Alchemy to send a transaction, sign anything, or hold a key. That is consistent with everything covered in files 02 through 05, this company's writing path is entirely handled by the user's own wallet plus `ethers.js`, and Alchemy's role here is strictly the read side, answering "what already exists on chain" quickly and cheaply, without this backend running its own infrastructure to know the answer.

## Where to go next

[07-moralis-and-cross-chain-nft-lookups.md](07-moralis-and-cross-chain-nft-lookups.md) covers the second data provider this codebase leans on for the same kind of question, and shows where its coverage overlaps with Alchemy's and where it is still the only path available for a particular chain.
