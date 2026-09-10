# 07. Domain Tenure, Tracking How Long Someone Has Really Held a Domain

## A newer, smaller, event driven system

`domain-tenure` is the newest looking sub module in this folder, and it solves a problem that only exists because a domain here is a blockchain asset rather than a database row, how do you know, with real confidence, when a user actually first came to own a given domain, and whether it has changed hands since. This matters for things like proving provenance, showing an accurate acquisition date on a portfolio, or detecting a domain that was quietly transferred to a different wallet.

## The entity

```ts
// src/components/domain/domain-tenure/entity/domain-tenure.entity.ts
@Entity({ name: 'tbl_domain_tenure' })
@Unique('tbl_domain_tenure_unique_constraint', ['domainName', 'domainProvider', 'userId'])
export class DomainTenureEntity extends BaseEntity {
    @Column({ nullable: false }) domainName: string;
    @Column({ nullable: false }) domainProvider: string;
    @Column({ nullable: false }) userId: string;
    @Column({ nullable: true }) token_id: string;
    @Column({ nullable: true }) firstSeenWalletAddress: string;
    @Column({ type: 'timestamptz', nullable: false }) firstSeenAt: Date;
    @Column({ nullable: true }) currentOwnerAddress: string;
    @Column({ type: 'timestamptz', nullable: true }) ownershipTransferredAt: Date;
    // false when firstSeenAt was set from system time because acquiredAt was unavailable.
    // The enrichment service updates this to true after verifying via getAssetTransfers.
    @Column({ type: 'boolean', default: false }) isOnChainDateVerified: boolean = false;
}
```

The comment on `isOnChainDateVerified` is the key to the whole design, this table is willing to record a best guess immediately, using whatever acquisition timestamp Alchemy's own NFT metadata happens to report, but it explicitly tracks whether that guess has actually been confirmed against real transfer history yet, and a separate background process exists purely to go do that confirmation later, asynchronously, without making the user wait for it.

## Two services, split by responsibility

`DomainTenureService` is small and reactive, it does not get called directly by any controller at all, it only exists to listen for the domain sync event described in the previous note:

```ts
// src/components/domain/domain-tenure/domain-tenure.service.ts
@OnEvent('domain.sync.completed', { async: true })
async handleDomainSyncCompleted(payload: DomainSyncCompletedEvent): Promise<void> {
    await this.domainTenureRepo.upsertTenureRecords(payload.domains, payload.userId);
    this.eventEmitter.emit('domain.tenure.enrich', { userId: payload.userId, domains: payload.domains });
}
```

Every time `refresh_domain` finishes syncing a user's wallets (the previous note), this quietly upserts a first pass tenure record for every domain found, and then immediately fires a second event, `domain.tenure.enrich`, handing off to the second service to go verify those dates properly.

`DomainTenureEnrichmentService` is where the real work happens, and it goes back to Alchemy again, but for a different purpose than the ownership sync did, this time asking specifically for the full ERC721 transfer history into the user's wallet, so it can find the true, earliest on chain timestamp a given token actually arrived there:

```ts
// src/components/domain/domain-tenure/domain-tenure-enrichment.service.ts
const PROVIDER_NETWORK_MAP: Record<string, Network> = {
    ENS: Network.ETH_MAINNET,
    UD: Network.MATIC_MAINNET,
    UDBASE: Network.BASE_MAINNET,
    Arbitrum: Network.ARB_MAINNET,
    BinanceSmartChain: Network.BNB_MAINNET,
};

@OnEvent('domain.tenure.enrich', { async: true })
async handleEnrichment(payload: TenureEnrichEvent): Promise<void> {
    const unverified = await this.domainTenureRepo.findUnverifiedByUserId(payload.userId);
    if (!unverified.length) return;
    const byNetwork = this.groupByNetwork(unverified);
    for (const [network, records] of byNetwork.entries()) {
        const alchemy = this.getAlchemy(network);
        const walletAddress = records[0].currentOwnerAddress;
        const registryAddresses = this.getRegistryAddresses(payload.domains, records);
        const transferMap = await this.fetchEarliestTransfers(alchemy, walletAddress, registryAddresses);
        ...
    }
}
```

Only records not yet verified are picked up, they are grouped by which blockchain network they actually live on so this only makes one Alchemy call per chain rather than one per domain, and `fetchEarliestTransfers` pages through Alchemy's `getAssetTransfers` API sorted in ascending order, converting hexadecimal token ids to decimal along the way, until it has the earliest confirmed transfer timestamp for every relevant token. A caught, specifically identified failure mode is worth noticing:

```ts
const is403 = networkError?.message?.includes('403') || networkError?.message?.includes('not enabled');
if (is403) {
    this.logger.warn(`Asset transfers not enabled for network "${network}" on this Alchemy app. Enable it at https://dashboard.alchemy.com and re-trigger a sync to backfill.`);
} else {
    this.logger.error(`Enrichment failed for network "${network}"`, networkError);
}
```

This is a real, previously hit operational issue, Alchemy gates the asset transfer API per network per API key, and this service was written specifically to keep going and enrich whatever chains it can rather than one disabled network silently breaking enrichment for every other chain a user's domains happen to live on.

## Why this exists as its own, separate, delayed pipeline

The honest reason this whole system is built as two chained events rather than one synchronous call inside `refreshDomainDetailData` is cost and latency. The refresh endpoint already makes six or seven blockchain API calls per user, as the previous note describes in detail, and is already rate limited specifically because of that cost. Adding a full transfer history lookup per chain on top of that, synchronously, on every single refresh, would make an already expensive endpoint dramatically more expensive and slower for no benefit the user would notice in the moment. Splitting it into a fire and forget background event means the user gets their updated domain list back immediately with a reasonable best guess date, while the more expensive, more accurate verification happens quietly afterward and simply updates the record once it is done.
