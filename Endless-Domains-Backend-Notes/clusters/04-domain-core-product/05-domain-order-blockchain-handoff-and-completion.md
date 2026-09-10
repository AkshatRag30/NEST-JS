# 05. Where This Module Hands Off to a Blockchain, and How the Order Gets Finished

## Two distinct handoff points

There are exactly two moments where this module's job ends and a chain specific integration folder's job begins, and they are worth being precise about, since the task of registering a domain on a real blockchain is deliberately kept out of this folder entirely.

## Handoff one, building the pending entity before any money moves

Before any payment happens, `DomainOrderService.orderDomain` calls out to a small dedicated class, `DomainOrderReturnDomainOrderEntity`, whose entire job is constructing the `DomainOrderEntity` and its nested `DomainDetailEntity` and `DomainMintEntity` rows, in memory, not yet touching a blockchain at all:

```ts
// src/components/domain/domain-order/domain-order-return-domain-order-entity.ts
private createOrderEntity(domainProvider: DomainProvider, blockchain: Blockchain, domainOrderDtoList: DomainInfoDto[]): DomainOrderEntity {
    ...
    const domainMintEntity = new DomainMintEntity();
    domainMintEntity.mintStatus = domainDetailDto.event ? MintStatus.COMPLETED : MintStatus.PENDING;
    domainMintEntity.domainProvider = domainProvider;
    domainMintEntity.blockchain = blockchain;
    ...
    const domainDetailEntity = new DomainDetailEntity();
    domainDetailEntity.ownerAddress = null;
    domainDetailEntity.resolver = null;
    domainDetailEntity.resolution = '{}';
    domainDetailEntity.registryAddress = null;
    domainDetailEntity.node = null;
    ...
}
```

Every field that can only be known once a real blockchain transaction has actually happened, `ownerAddress`, `registryAddress`, `node`, `resolver`, is explicitly set to `null` or an empty object here, this is the row as it looks the instant before anything on chain has occurred, mint status `PENDING`, waiting.

There is one deliberate exception, worth understanding closely, for Unstoppable Domains' "event" domains:

```ts
/**
 *  For Event Page Following actions followed
 *  1. check if any domain info dto has event flag true (...)
 *  2. if yes then
 *      2.1. check status of domain name in tbl_custom_domain
 *          2.1.1. if sold then throw error
 *          2.1.1. if available then    
 *              2.1.1.1 check in endless wallet
 *                          2.1.1.1.1. if yes then create domain detail entity as before but mint entity status will be COMPLETE ...
 *                          2.1.1.1.2. if no then update sold in tbl_custom_domain and throw error.
 */
```

Some UD domains sold during a promotional event are names this company already owns and holds in its own wallet ahead of time, rather than names that need to be freshly registered on Unstoppable Domains' registrar at purchase time. For those, `mintStatus` is set straight to `COMPLETED` and the whole later "wait for a blockchain transaction to confirm" step never happens, the domain is transferred out of the company's own wallet instead. `checkIfDomainNameAreInTblCustomDomain` and `updateStatusCustomDomainName` are the calls into the `custom-domain` component that track exactly which of these pre held names are still available versus already sold, and this exact spot is a real handoff to that other module, not something this note needs to explain further since it belongs to a different slice of the codebase.

## Handoff two, the actual on chain registration, and the way it comes back

For every provider except that special event case, the domain is not really the buyer's yet the moment they pay, minting still has to happen. This module exposes exactly two endpoints for that, and it is the frontend and the buyer's own connected wallet, not this backend, that actually talks to the blockchain in between them.

```ts
// domain-order.controller.ts
@Post('/request-register')
public async web3RequestRegister(@Req() req: Request, @Body() requestRegister: RequestRegisterDto): Promise<Response> { ... }

@Post('/register')
public async web3Register(@Req() req: Request, @Body() web3RegisterTransaction: Web3RegisterTransactionDto): Promise<Response> { ... }
```

`web3SaveRequestRegister` is called first, right after the user's wallet has signed and submitted the initial registration transaction. This method does not talk to any blockchain itself, it records that a transaction has been requested, using the injected `Web3TransactionServiceInterface` (a shared transaction logging service covered elsewhere), flips the order to `PROCESSING`, and flips every mint record inside it to `PROCESSING` as well. It also fires an event driven email through `EventEmitter2` letting the buyer know their order is being processed, `DOMAIN_ORDER_PROCESSING`.

`web3SaveRegisterTransaction` is the one that finally completes everything, called once the frontend has confirmed the wallet's transaction actually landed on chain and has the resulting transaction hash in hand:

```ts
const blockchainExplorer = getBlockchainExplorerByNetworkAndDomainProvider(domainProvider, web3RegisterTransactionDto.transactionHash, this.BLOCKCHAIN_NETWORK);
...
await this.domainOrderRepo.updateOrderStatusSecretByUserIdAndOrderId(userId, web3RegisterTransactionDto.orderId, DomainOrderStatus.COMPLETED);
...
const registrarAddress = await this.domainProviderSmartContractRepoInterface.getContractAddressByParams(domainProvider, blockchainNetwork, SmartContractRegistrarEnum.REGISTRAR);
...
domainDetailIdList.domainDetailList.map((domainDetail) => {
    ...
    domainDetail.domain_transfer_status = DomainDetailTransferEnum.COMPLETE;
    domainDetail.ownerAddress = user.walletAddress;
    domainDetail.registryAddress = registrarAddress.registrar;
    domainDetail.domainMintList.map((domainMint) => {
        domainMint.mintStatus = MintStatus.COMPLETED;
        domainMint.blockchainExplorer = blockchainExplorer;
        domainMint.transactionId = web3RegisterTransactionDto.transactionHash;
    });
});
await this.domainOrderRepo.save(domainDetailIdList);
```

This is the exact moment a domain record in this database goes from a placeholder with every blockchain field null to a fully real, owned domain, `ownerAddress` is set to the buyer's own wallet address, `registryAddress` comes from a lookup against `DomainProviderSmartContractRepoInterface`, a small table this codebase keeps of which smart contract address acts as the registrar for each provider on each network (mainnet versus testnet), and every mint record tied to this order flips to `COMPLETED` carrying the real transaction hash and a direct explorer link. Nowhere in this call does this backend itself submit a blockchain transaction, sign anything with a private key, or talk to a node directly, the actual chain interaction already happened in the user's own wallet, this backend's job here is purely to record the outcome faithfully once it is told the transaction hash. That signing and submission logic lives in the chain specific integration folders this note deliberately does not go into.

Right after completion, this same method also generates two separate invoices, a sales invoice through `SalesInvoiceServiceInterface` and a purchase invoice through `PurchaseInvoiceServiceInterface`, and emits `DOMAIN_ORDER_COMPLETED` so the buyer gets a confirmation email, both real, visible side effects of a successful mint that are easy to miss on a first read of this method.

## `DomainMintService`, small and deliberately dumb

```ts
// src/components/domain/domain-mint/domain-mint.service.ts
async updateMintStatusByDomainDetailId(domainDetailId: GetDomainDetailIds[], mintStatus: MintStatus): Promise<void> {
    const domainDetailIdArray = domainDetailId.map((item) => item.domainDetailId).filter(Boolean);
    await this.domainMintRepo.updateMintStatusByDomainDetailId(domainDetailIdArray, mintStatus);
}
```

Everything in this service is a narrow, single purpose update against `tbl_domain_mint`, there is no branching logic of its own, it exists purely so the order service (and, in the past, an ENS specific flow now commented out in `web3SaveRegisterTransaction`) has a clean, testable place to flip mint statuses in bulk by domain detail id, without every caller needing to know the shape of the mint repository directly.
