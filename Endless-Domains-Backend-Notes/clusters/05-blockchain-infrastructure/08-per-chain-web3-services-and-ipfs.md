# 08. The Per Chain web3 Services, web3-transaction, and IPFS

## What is left in the `web3` folder

Files 02 through 07 already covered most of the interesting mechanics in this cluster, so this file rounds out the rest of the `web3` folder, the small per chain services (`ens-web3`, `arb-web3`, `ud-web3`, `bnb-web3`), the `web3-transaction` module, and `ipfs`, three things that share a folder but do genuinely different jobs.

## The per chain web3 services, a small, repeated pattern

`ens-web3.service.ts`, `arb-web3.service.ts`, and `ud-web3.service.ts` (file 02 already showed the ENS and Arbitrum versions reading a contract directly) all exist to answer two narrow questions for one specific chain, "is this domain name available to register" and "what is this token's metadata." `UDWeb3Service` is the simplest of the three, and worth reading precisely because of what it does not do:

```ts
async getMetaData(tokenId: string): Promise<UDMetaDataInterface> {
    const axiosResponse = await httpClient.get(this.getBcNetwork(this.BLOCKCHAIN_NETWORK).metaDataUrl + tokenId + `?randomId=${generateRandomUUID()}`);
    return this.toUDMetaDataInterface(axiosResponse.data);
}
```

There is no `web3.js` import anywhere in this file at all, no contract, no ABI. Unstoppable Domains exposes its own plain HTTP metadata API, so this service just calls that directly with `httpClient` (a shared axios wrapper used throughout this project), appending a random query parameter, `randomId`, purely to defeat any HTTP caching layer that might otherwise serve stale metadata. Compare that against `EnsWeb3Service` and `ARBWeb3Service`, both of which do construct a `web3.eth.Contract` and call `.methods.available(domainName).call()` directly against a real contract, because ENS and this company's own Arbitrum registrar do not offer an equivalent, ready made HTTP API for that specific check, so this backend has to ask the contract itself. Which approach a given chain's service uses is entirely dictated by what that chain's ecosystem actually offers, not by any preference in this codebase.

Every one of these services resolves its own registrar contract's address through `DomainProviderSmartContractRepoInterface`, the same repository seen throughout `alchmey` and `moralis`, rather than hard coding an address in the TypeScript file, letting this company update a registered contract address (say, after a redeploy or a migration to a new registrar version) by changing a database row instead of shipping new code. `getBcNetwork`, repeated with only cosmetic differences across every one of these services, switches between `BlockchainNetwork.MAINNET` and `BlockchainNetwork.TESTNET` to pick the right metadata URL or contract address for whichever environment the server is actually running in.

## `web3-transaction`, a name that undersells what it actually is

Despite living in the `web3` folder, `web3-transaction.service.ts` never touches a blockchain at all. Every one of its methods, `createTransactionLogs`, `getTransactionLogByOrderId`, `getAllTransactionLogs`, `exportTransferDomainsByDateRange`, just reads from or writes to `Web3TransactionRepoInterface`, a normal TypeORM backed repository over a normal Postgres table:

```ts
async createTransactionLogs(web3CreateRequestRegisterDto: Web3TransactionLogsDto): Promise<Web3TransactionLogsResponseDto> {
    const data = await this.web3Repo.save(web3CreateRequestRegisterDto);
    return data;
}
```

This module exists to give the rest of the domain ordering flow (covered in its own cluster of notes) a durable, queryable record of every blockchain related transaction this company's users have gone through, transaction hashes, order IDs, transaction types and statuses, kept in this company's own database rather than only ever existing on chain. It is worth remembering this pattern by name, "on chain" data (the real transaction, permanently recorded on whatever blockchain it happened on) and "off chain" data (this company's own record of that transaction, kept for fast lookups, admin reporting, and export) are two different things that need to be kept in sync, and a huge amount of Web3 backend code, this module included, exists purely to do that syncing and bookkeeping, not to talk to a chain directly at all.

## `ipfs`, a module that is not really about blockchains either

`ipfs.service.ts` is the largest file in the `web3` folder, and reading it carefully, it is not primarily a blockchain integration, it is a file upload and pinning service. IPFS, the InterPlanetary File System, is a separate, content addressed storage network, and Pinata (wrapped here as `PinataService`, from a different component folder, `ipfs-server`) is a company that runs IPFS infrastructure so that a file, once pinned, stays reliably available. `uploadWebsiteToIPFS` takes a set of uploaded files (a customer's own static website, given the `domainName/assets/image.jpg` style paths referenced in its comments), packages them into a `FormData` payload, and hands them to Pinata:

```ts
const pinataResponse = await this.pinataService.pinToIPFS(formData, 'pinataMetadata', metadata);
...
const ipfsDto = new CreateIpfsDto();
ipfsDto.ipfsHash = pinataResponse;
...
return this.toIPFSResponseInterface(ipfsDataStored, metaDataStored, gigapub_id);
```

The IPFS hash that comes back is this company's real, load bearing content address, the same shape of value as the `ipfs://` URI produced by `NftCollectionService.uploadMetadataToIpfs` in file 04, and it is what gets stored, alongside the rest of that request's metadata, in this company's own `tbl_ipfs`-backed table so a domain's website content can be found again later. The connection back to blockchains is real but indirect, a domain that lives as an NFT can point, through its on chain metadata or resolver record, at an IPFS hash like this one, which is how a Web3 domain can resolve to an actual, decentralized website rather than a server this company controls, but the module itself, the file upload, the Pinata call, the database bookkeeping, is doing ordinary backend work, not talking to a chain.

## Where to go next

[09-listener-what-it-really-is.md](09-listener-what-it-really-is.md) closes this cluster with the one finding in it that most contradicts a first guess based on a folder's name alone, and shows the evidence for what `listener` actually turned out to be once its code was actually read.
