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

## Update from the October 2026 uat pull

Three things changed in the `alchmey` folder in this pull, and one deliberately did not.

The thing that did not change is the `getMetaData` excerpt above. It still uses `BigNumber.from(tokenId)`, and that is correct even after the whole repository moved to ethers v6 in commit `71fd2fec`, because the import at `src/components/alchmey/arbAlchmey/arb-alchmey.servers.ts` line 11 is `import { BigNumber } from "@ethersproject/bignumber"`, the standalone v5 package, not the `ethers` package. `alchemy-sdk` 3.x is itself built on the v5 `@ethersproject/*` packages, so its `getNftMetadata` signature expects a v5 `BigNumberish`, and handing it a v5 `BigNumber` is the safest choice. Commit `26b1f0e4` added `src/components/alchmey/arbAlchmey/arb-alchmey.servers.spec.ts` (one test) to pin exactly this. It builds the service with a fake `ConfigService` (`{ get: jest.fn((key) => env[key]) }`), swaps `(service as any).alchemy` for an object whose `nft.getNftMetadata` is a `jest.fn()`, calls `getMetaData('42', '0xContractAddress')`, and asserts that the second argument passed to Alchemy satisfies `BigNumber.isBigNumber(...)`, is not a `bigint`, is not a string, and stringifies to `'42'`. The point is to stop a future well meaning "finish the v6 migration" commit from replacing `BigNumber.from(tokenId)` with `BigInt(tokenId)`, which would pass a type `alchemy-sdk` was never written for. The honest risk here is packaging, not code: `@ethersproject/bignumber` is not declared in `package.json`, it is only installed because `alchemy-sdk` depends on it (`package-lock.json` line 8715). If it ever stops being hoisted, line 11 fails at boot.

The second change is the same kind of guard for keccak hashing. `src/components/alchmey/fetch-expiry/fetch-expiry.service.ts` line 416 and `src/components/ud-integration/ud-integration.service.ts` line 1248 each keep their own private `keccak256` helper built from `js-sha3` plus `arrayify` from `@ethersproject/bytes`. The new `src/components/alchmey/fetch-expiry/keccak256-cross-agreement.spec.ts` (one test, commit `26b1f0e4`) constructs both services with hand written mocks, calls both private helpers with `'test'`, and asserts they return the identical string and that it matches `/^0x[0-9a-f]{64}$/`. Its doc comment files this under Sprint 11 as a low risk closure item of the ethers v6 migration. It proves the two copies agree with each other, it does not compare against a known external hash, so if both drifted together the test would still pass.

The third change is real new behaviour in `src/components/alchmey/bnb-alchmey/bnb-alchmey.serveice.ts`, added by commit `546e26e9` ("Implemented new poller system", 21 September 2026). Before this pull, the BNB expiry came only from the space.id GraphQL indexer. Now `getDomainExpiry` (lines 164 to 213) asks the base registrar contract directly and only falls back to the indexer if that fails:

```ts
// src/components/alchmey/bnb-alchmey/bnb-alchmey.serveice.ts (lines 219 to 238)
private async getOnChainExpiry(label: string): Promise<string | null> {
    const blockchainNetwork =
        this.BLOCKCHAIN_NETWORK === 'Testnet'
            ? BlockchainNetwork.TESTNET
            : BlockchainNetwork.MAINNET;

    const registrar = await this.domainProviderSmartContractRepo.getContractAddressByParams(
        DomainProvider.BinanceSmartChain,
        blockchainNetwork,
        SmartContractRegistrarEnum.BASE_REGISTRAR,
    );
    if (!registrar?.registrar) {
        return null;
    }

    const contract = new this.web3Bnb.eth.Contract(BNBBaseRegistrarContract.abi as AbiItem[], registrar.registrar);
    const id = labelHash(label);
    const result: string = await contract.methods.nameExpires(id).call();
    return result || null;
}
```

The comment at lines 188 to 190 explains the motivation, "The SpaceID indexer's `expiration` can lag behind the base registrar after a renewal", which matters because the BNB renewal flow in [../06-chain-and-registrar-integrations/04-onchain-write-and-renewal-flows.md](../06-chain-and-registrar-integrations/04-onchain-write-and-renewal-flows.md) would otherwise show a user their old expiry right after paying to renew. Notice it uses web3.js (a new `this.web3Bnb` built from `BNB_RPC_URL` in the constructor at line 45), consistent with the "web3 reads, ethers writes" split in file 02, but it borrows `labelHash` from `src/components/eth-domain-renewal/utils/label-hash.util.ts`, which is an ethers v6 function (`ethers.keccak256(ethers.toUtf8Bytes(label))`). A registrar token id for a label is `uint256(keccak256(label))`, so passing that 32 byte hex string as the `uint256` argument is correct.

There are three honest risks in this new code. First, `return result || null` at line 237 treats the string `"0"` as a real expiry, because a non empty string is truthy in JavaScript. A registrar's `nameExpires` returns `0` for a label it has never registered, so if `meta.name` is ever not the bare label (for example if the indexer returns it with the `.bnb` suffix, or the DB row points at the wrong registrar for the current `BLOCKCHAIN_NETWORK`), the indexer's correct value is overwritten with `"0"`, which every later consumer reads as 1 January 1970, an already expired domain. Checking `result && result !== '0'` would close that. Second, the indexer's `expiration` and the contract's `nameExpires` are assumed to be in the same unit (unix seconds) and the code never normalises either, so if the indexer ever returned a different format the two sources would disagree silently. Third, `filterMoralisData` calls `getDomainExpiry` inside a sequential `for` loop (line 118), and every iteration now adds one database query plus one BNB RPC call on top of the GraphQL call, so a wallet holding thirty BNB domains does ninety awaited network round trips one after another during a refresh.

The fourth change is in `src/components/alchmey/freename-alchmey/freename-alchmey.serveice.ts`, from commit `96f221ec` ("fixed the reported bugs", 30 September 2026). Line 123 now passes `{ headers, timeout: EXTENDED_HTTP_TIMEOUT_MS }` (45 seconds, from `src/@core/utils/http-client.util.ts` line 19) instead of relying on `httpClient`'s shared 8 second default. The comment at lines 116 to 122 is unusually candid about why: a timeout there used to silently wipe the user's already synced Freename rows on the next refresh, because of the delete then insert behaviour of `bulkUpdateDomainDetailBlockChain`. In other words, a slow Freename page made the fetch fail, the refresh treated the user as owning zero Freename domains, and the delete then insert sync erased them. The longer timeout makes that much less likely but does not remove the root cause, a failed fetch is still indistinguishable from "owns nothing" further down. The existing spec `freename-alchmey.serveice.spec.ts` was updated to add `EXTENDED_HTTP_TIMEOUT_MS: 45000` to its `jest.mock` of the HTTP client and to assert the new `timeout` option in the `toHaveBeenCalledWith` expectation. The codebase wide migration story is in [../12-marketplace-v2/12-ethers-v6-migration-and-the-new-test-suite.md](../12-marketplace-v2/12-ethers-v6-migration-and-the-new-test-suite.md).
