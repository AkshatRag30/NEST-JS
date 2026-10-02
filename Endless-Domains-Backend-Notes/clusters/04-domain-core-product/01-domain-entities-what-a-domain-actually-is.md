# 01. The Core Entities, and What Actually Makes a Domain Here Different

## Two tables, not one, for the same idea

The first thing worth noticing, before reading a single field, is that this module keeps two separate entities that both look like "a domain a user owns," `DomainDetailEntity` and `DomainDetailBCEntity`. A comment directly on the first one explains why:

```ts
// src/components/domain/domain-detail/entity/domain-detail.entity.ts
/*
    Note: This table is used to store data of domain details that are bought using Endless domain. This is done to store historic data of domain bought using endless domains.
    Here the order table, mint table and domain details have many to one or one to manny relationship.  
*/
@Entity({ name: 'tbl_domain_detail' })
export class DomainDetailEntity extends BaseEntity {
```

`tbl_domain_detail` is the transactional record of a purchase made through this company's own checkout, it is the row that gets created the moment someone puts a domain in their cart and pays for it, and it stays linked to the order and mint records that produced it. `tbl_domain_detail_bc` (the `BC` stands for blockchain) is a completely different kind of record, a synced snapshot of whatever domains a user's actual wallet address owns on chain right now, refreshed by directly querying blockchain indexing services. A user can own a domain in `tbl_domain_detail_bc` that this company never sold them at all, because they bought it somewhere else and simply connected the wallet that holds it. This split matters a lot once you are reading the search, order, and detail services, because each one talks to a different one of these two tables depending on whether it cares about "what did we sell" or "what does this wallet actually hold right now."

## `DomainDetailEntity`, field by field

```ts
@Column({ nullable: true }) domainId: string;
@Column({ nullable: false }) domainName: string;
@Column({ nullable: false }) domainProvider: string;
@Column({ nullable: true }) ownerAddress: string;
@Column({ nullable: true }) resolver: string;
@Column({ nullable: true }) resolution: string;
@Column({ nullable: true }) blockchain: string;
@Column({ nullable: true }) projectedBlockchain: string;
@Column({ nullable: true }) registryAddress: string;
@Column({ nullable: true }) networkId: string;
@Column({ nullable: true }) node: string;
@Column({ nullable: false, default: 0 }) price: number;
@Column({ nullable: true }) noOfYear: number;
@Column({ nullable: true }) domainPurchaseExpireDate: Date;
@Column({ nullable: false, default: 'Pending', comment: '...' }) domain_transfer_status: string;
```

`domainId` is whatever id the external registrar or blockchain uses for this name, separate from this row's own database `id`. `domainProvider` is one of a fixed set of string values, `UD` for Unstoppable Domains, `ENS`, `Arbitrum`, `BinanceSmartChain`, `Bonfida` for Solana names, `Tezos`, `Aptos`, `Ton`, `Starknet`, `Box`, and `Freename`, and it is the single most important field in this whole module, almost every service branches its behavior on this value. `ownerAddress` is a wallet address, not a username or an email, this is the first concrete sign of what makes this different from a normal domain purchase, ownership is recorded as a blockchain address, the same kind of address used to hold cryptocurrency. `resolver` and `resolution` describe how the domain actually resolves to content or an address on chain, `resolution` is stored as a JSON string (you can see `domainDetailEntity.resolution = '{}';` used as a default when a fresh order is created, and later parsed back with `JSON.parse` when returned to the frontend). `registryAddress` and `node` are two more concepts that have no equivalent at all in a traditional DNS registrar, `registryAddress` is the address of the actual smart contract that acts as the registry for this domain, and `node` is the ENS specific concept of a hashed identifier for the name within that registry's namespace. `noOfYear` and `domainPurchaseExpireDate` exist because several of these providers, ENS among them, sell domains as time bound leases the same way a normal registrar does, you are renting the name for a number of years, not owning it outright forever, whereas others (Unstoppable Domains, Bonfida) are sold as a one time purchase with no expiry at all, which is exactly why `noOfYear` is set to `null` for those providers in the order creation code you will see in the next note. `domain_transfer_status` tracks whether the actual on chain transfer of the domain to the buyer's wallet has completed yet, separate from whether the order itself is paid for.

The relationships on this entity tie the whole purchase story together:

```ts
@OneToMany(() => DomainMintEntity, (domainMint) => domainMint.domainDetail, { cascade: true })
domainMintList: DomainMintEntity[];

@ManyToOne(() => DomainOrderEntity, (domainOrderEntity) => domainOrderEntity.domainDetailList)
domainOrder: DomainOrderEntity;
```

One order can contain several domain names (a user buying three names in one checkout), and each domain name can have more than one mint record, because a mint can fail and be retried, or in some providers, a purchase involves more than one on chain step.

## `DomainDetailBCEntity`, the on chain mirror

```ts
// src/components/domain/domain-detail/entity/domain-detail_bc.entity.ts
@Entity({ name: 'tbl_domain_detail_bc' })
@Unique('tbl_domain_detail_bc_unique_constraint_domain_name_and_token_id_and_userId', ['domainName', 'token_id', 'userId'])
@Index('idx_domain_detail_bc_domain_name', ['domainName'])
@Index('idx_domain_detail_bc_user_id', ['userId'])
@Index('idx_domain_detail_bc_domain_provider', ['domainProvider'])
export class DomainDetailBCEntity extends BaseEntity {
    ...
    @Column({ nullable: true }) token_id: string;
    @Column('jsonb', { nullable: true }) domainMint: JSON;
    @Column({ nullable: true }) expiryDate: string;
    @Column({ nullable: true }) zone_uuid: string;
}
```

Most fields mirror `DomainDetailEntity` exactly, since it represents the same underlying idea, but a few differences reveal what this table is actually for. `token_id` is present here in a way it is not on the transactional table, because this row was built by reading an actual NFT off a wallet, and every domain NFT has a token id inside its collection contract, this is the literal proof that a domain here is implemented as an NFT (a unique, ownable token) sitting in a smart contract, not a row in some company's private registrar database. `expiryDate` is stored as a raw string here, deliberately, because it comes straight out of whatever the blockchain indexer returns and different chains format it differently, you will see later that reading it back out requires checking whether it is a bare epoch number using a regular expression before doing any date math on it. The unique constraint across domain name, token id, and user id is what lets this table be safely rebuilt from scratch every time a user hits refresh, upserting rather than duplicating rows for domains they already had on file.

## `DomainOrderEntity`, the actual purchase

```ts
// src/components/domain/domain-order/entity/domain-order.entity.ts
@Entity({ name: 'tbl_domain_order' })
export class DomainOrderEntity extends BaseEntity {
    @Column({ nullable: false }) orderNumber: string;
    @Column({ nullable: false }) domainProvider: string;
    @Column({ default: 'Pending' }) orderStatus: string;
    @Column({ nullable: false }) totalCost: number;
    @Column({ nullable: true }) promoValue: number;
    @Column({ default: false }) promoApplied: boolean;
    @Column({ nullable: true }) promoCodeUsed: string;
    @Column({ nullable: true }) paymentMethod: string;
    @Column({ nullable: true }) paymentClientId: string;
    @Column({ nullable: true }) paymentIntentId: string;
    @Column({ nullable: true }) cryptomusUuid: string;
    @Column({ nullable: true }) transactionSecret: string;
    ...
    @Column({ name: 'txn_fee', nullable: false, type: 'decimal', precision: 5, scale: 2, default: '0.00' }) txnFee: number;
    @Column({ default: false, comment: '...' }) event: boolean;
    @Column({ type: 'boolean', default: false }) emailReminderSent: boolean;
}
```

This is a normal looking e commerce order table in most respects, an order number, a status, a total cost, coupon fields. What is not normal is `paymentIntentId` sitting right next to `transactionSecret`, because this order can be paid for in two completely different ways that both need tracking on the same row, a card through Stripe (`paymentClientId`, `paymentIntentId`) or a cryptocurrency wallet transaction (`transactionSecret`, which holds a literal blockchain transaction hash once payment lands). `txnFee` is a decimal column specifically because blockchain transactions cost real network fees (commonly called gas) on top of the domain's own price, and this company passes an estimated share of that fee on to the buyer, calculated in the order service as shown in the purchase flow note. The `event` boolean, per its own comment, was a temporary flag for a Chinese New Year promotional batch of pre owned domain names, a good small example of a real business decision leaving a permanent trace in a schema long after the promotion itself ended.

## `DomainMintEntity`, tracking the on chain step itself

```ts
// src/components/domain/domain-mint/entity/domain-mint.entity.ts
@Entity({ name: 'tbl_domain_mint' })
export class DomainMintEntity extends BaseEntity {
    @Column({ nullable: true }) mintingId: string;
    @Column({ nullable: false }) domainProvider: string;
    @Column({ default: 'Pending' }) mintStatus: string;
    @Column({ nullable: false }) blockchain: string;
    @Column({ nullable: true }) blockchainExplorer: string;
    @Column({ nullable: true }) transactionId: string;
    @ManyToOne(() => DomainDetailEntity, (domainDetailEntity) => domainDetailEntity.domainMintList)
    domainDetail: DomainDetailEntity;
}
```

"Minting" is the blockchain term for the act of creating a brand new token, in this case the domain NFT itself, and this table exists purely to track that one specific step's lifecycle, separate from the order's own payment status. `mintStatus` moves through `Pending`, `Processing`, `Completed`, `Failed`, and `On_Hold` (defined in `src/@core/common/enum/mint-status.enum.ts`), and `blockchainExplorer` stores a direct link to a site like Etherscan where a buyer can go look at their own transaction and see it confirmed on a public, permanent ledger, something that has no equivalent at all in a traditional domain purchase, where you simply trust the registrar's own database.

## `DomainProviderAndTldsEntity` and `DomainTenureEntity`, briefly

Two more entities worth knowing the shape of here, covered in full in their own notes. `DomainProviderAndTldsEntity` (`tbl_domain_provider_and_tlds`) is a tiny reference table, just `domain_provider`, `abbreviation`, and `tld`, that answers the single most repeated question in this whole module, given a TLD like `.crypto`, which provider actually issues it. `DomainTenureEntity` (`tbl_domain_tenure`) is a newer table that tracks, per user and per domain, `firstSeenWalletAddress`, `firstSeenAt`, `currentOwnerAddress`, and `ownershipTransferredAt`, essentially a provenance record of how long someone has actually held a domain and whether it changed hands, verified against real blockchain transfer history rather than trusted blindly, covered in full in note seven.

## What actually makes a domain here different from a GoDaddy domain

Every field above adds up to one real answer, grounded in what these entities actually store rather than a general explanation of Web3. A domain from GoDaddy is a row in GoDaddy's own private database, you are renting the right to have that row point your name at a server, and GoDaddy is the only party on earth who can tell you who owns it or move it to someone else. A domain here, looking at what `ownerAddress`, `registryAddress`, `node`, and `token_id` actually store, is a non fungible token sitting inside a smart contract that this company does not privately control, ENS's contracts, Unstoppable Domains' contracts, and so on. Ownership is not a database row, it is whichever wallet address the blockchain itself says holds that token, which is exactly why `DomainDetailBCEntity` exists as a separate table at all, this company has to go ask each blockchain what a wallet actually owns, the same way anyone else with a blockchain explorer could, rather than simply looking the answer up in its own private database the way GoDaddy would. It is also why a purchase here produces both an order row and a mint row, buying the domain and creating (minting) the actual token that represents it are two distinct steps with two distinct failure states, and why some of these domains expire on a lease timer (`noOfYear`, `domainPurchaseExpireDate`) while others, once minted, are owned outright forever with no renewal at all, a genuine behavioral split between providers that this module's code has to branch on constantly rather than a detail this company invented.

## Update from the October 2026 uat pull

One repository method on `DomainDetailBCEntity`'s repository changed shape in commit `b8c0f074` ("Added new table realated changes", 25 September 2026), and it is a good small lesson in why `token_id` alone is not an identity. Before the pull, `findByTokenId(tokenId)` in `src/components/domain/domain-detail/domain-detail-bc.repo.ts` was a plain `this.repo.findOne({ where: { token_id: tokenId } })`. Nothing in the codebase called it at the baseline commit. Now it takes a second argument and uses the query builder:

```ts
// src/components/domain/domain-detail/domain-detail-bc.repo.ts (lines 501 to 508)
async findByTokenId(tokenId: string, registryAddress: string): Promise<DomainDetailBCEntity | null> {
    return this.repo
        .createQueryBuilder('d')
        .where('d.token_id = :tokenId', { tokenId })
        .andWhere('LOWER(d."registryAddress") = LOWER(:registryAddress)', { registryAddress })
        .andWhere('d."isDeleted" = false')
        .getOne();
}
```

The new doc comment on the interface at `src/components/domain/domain-detail/interface/domain-detail-bc.repo.interface.ts` line 39 states the reason precisely: "without it, a tokenId that collides across two different collections/chains can match the wrong row." Remember from the entity section above that a token id is only unique inside its own collection contract. Token id `42` on the ENS base registrar and token id `42` on a UD contract are two completely unrelated domains, so a lookup by `token_id` alone could hand back the wrong domain, and on a table holding every chain at once that is not hypothetical. Scoping by `registryAddress` (the collection contract address) fixes that, `LOWER(...)` on both sides makes the match immune to EIP 55 checksum casing (the same address can be stored as `0xAbC...` or `0xabc...`), and the `isDeleted = false` clause stops a soft deleted row from being returned, which the old `findOne` did not filter either.

The only caller is the new marketplace v2 order validation at `src/components/marketplacev2/order/order.service.ts` line 370, which passes `this.chainConfig.domainNftAddress`, so this is really infrastructure for [../12-marketplace-v2/03-seaport-primer-and-the-order-entity.md](../12-marketplace-v2/03-seaport-primer-and-the-order-entity.md). Two honest caveats. First, wrapping the column in `LOWER(...)` means Postgres cannot use a plain index on `"registryAddress"`, and the existing indexes on this table are on `domainName`, `userId` and `domainProvider` only, so this lookup is a scan filtered by `token_id`; fine at today's size, worth an expression index (`CREATE INDEX ... ON tbl_domain_detail_bc (token_id, LOWER("registryAddress"))`) if the table grows. Second, the unique constraint on this table is across `domainName`, `token_id` and `userId`, so the same token can legitimately appear in two rows for two different users if a previous owner's row has not yet been cleaned up by their next refresh. `getOne()` with no `ORDER BY` then returns whichever row Postgres finds first, which could be the stale previous owner. Callers that care about the current owner should check ownership on chain rather than trust this row, and the v2 order service does exactly that: it reads `ownerOf(tokenId)` from the contract first (line 357) and uses this row only to confirm the domain name (line 371), which a stale row would still get right.
