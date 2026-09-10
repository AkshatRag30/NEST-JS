# 03. The Secondary Marketplace: Listing and Buying an Already Owned Domain

## A genuinely different business from the cart and checkout flow

Everything in `01` and `02` is about a user registering a brand new domain name for the first time, paid through a card or a crypto payment gateway. This file covers a different product entirely, one user reselling a domain they already own directly to another user, priced and paid for on chain, with no Stripe, CoinGate, or Cryptomus involved at all. The two flows share a database (and a couple of entities lean on each other) but they are otherwise independent purchase pipelines, and it is worth keeping that straight while reading this file, `DomainListing` and `BuyDomainListing` are not the same thing as `DomainOrderEntity` from the primary flow.

## Listing a domain: `DomainListing`

```ts
// src/components/marketplace/domain-listing/entity/domain-list.entity.ts
@Entity('tbl_domain_listings')
export class DomainListing {
  @Column({ nullable: false }) tokenId: string;
  @Column({ nullable: false }) listingId: string;
  @Column({ nullable: false }) blockchainTxHash: string;
  @Column({ nullable: false }) blockchainStatus: string;
  @Column({ default: 'Pending' }) reconciliationStatus: string;
  @Column({ default: false }) listingStatus: boolean;
  @Column({ default: 'Available' }) purchaseStatus?: string;
  @Column({ enum: ['Active', 'Expired', 'Delisted'], default: 'Active' }) status: "Active" | "Expired" | "Delisted";
  @Column({ nullable: true }) isPremium?: boolean;
  @Column({ nullable: true }) isPromoted?: boolean;
  @Column({ nullable: true }) bulkGroupId?: string;
}
```

A row here does not represent "a listing the backend created," it represents "a listing the frontend already submitted directly to the marketplace smart contract, whose confirmation the backend is waiting on." `DomainListingService.create` is called only after the user's wallet has already sent the on chain `createListing` transaction (see `Web3Service.createListing` in `marketplace/web3/web3.service.ts` for the raw transaction building, though in practice the frontend, not this backend, is what actually signs and sends it against the live `NFTDomainsMarketplaceV5` contract). The backend's job at that point is to record the row as `blockchainStatus: 'Pending'`, `listingId` and `blockchainTxHash` already known, and wait for `transaction-cron` (covered in `04`) to confirm it actually landed. Four separate status fields track four separate questions about one row: `status` (is the listing itself active, expired, or delisted), `listingStatus` (a boolean mirror of whether the on chain listing is currently live), `purchaseStatus` (`Available`, `In Progress`, or `Sold`, tracking whether someone is mid purchase or has already bought it), and `blockchainStatus`/`reconciliationStatus` (the pending confirmation machinery itself, shared in shape with `BuyDomainListing`, see `04`).

Two details worth calling out. First, `assertNonZeroListingId`, called at the very top of both `create` and `createBulk` before anything else happens:

```ts
// src/components/marketplace/abi/listing-id.util.ts
export function assertNonZeroListingId(listingId: string): void {
  const value = BigNumber.from(listingId);
  if (value.isZero()) throw new BadRequestException('listingId 0 is not a valid listing');
}
```

The comment above it explains why this exists, the marketplace contract's own `tokenToListing` mapping defaults any uninitialized entry to `0`, so a `listingId` of `0` can never legitimately refer to a real listing, only to a bug or a spoofed request, and this guard rejects it before a single database write happens. Second, duplicate active listings are blocked at the application layer, not the schema, `create` looks up any existing row for the same `domainName` with `status: 'Active'` and throws a different message depending on whether the caller already owns that active listing or someone else does, a small but real UX distinction a frontend error message should preserve.

Bulk listing (`createBulk`) is the same idea for many domains at once under one shared `blockchainTxHash` and a generated `bulkGroupId`, which matters later because the reconciliation cron and the success email logic both key off `bulkGroupId` to send exactly one "your domains are listed" email per batch rather than one per domain.

## Buying a listed domain: `BuyDomainListing`

```ts
// src/components/marketplace/buy-domain/buy-domain.service.ts
async buyDomain(buyDomainDto: BuyDomainDto, userId: string) {
    const { listingId, buyer, price, domainName, domainId } = buyDomainDto;
    assertNonZeroListingId(listingId);

    const reservation = await this.domainRepo.update(
        { listingId, domainName, blockchainStatus: 'Success', listingStatus: true, purchaseStatus: 'Available' },
        { purchaseStatus: 'In Progress' }
    );

    if (reservation.affected !== 1) {
        throw new ConflictException('This domain has already been purchased or a purchase is already in progress');
    }

    const newBuyDomain = this.buyDomainRepo.create({ listingId, buyer, userId, pricePerToken: price, blockchainStatus: 'Pending', blockchainTxHash: buyDomainDto.blockchainTxHash, domainName, domainId });
    const saved = await this.buyDomainRepo.save(newBuyDomain);
    return saved;
}
```

The comment in the real code calls the first step exactly what it is, "an atomic conditional update," and it is a genuinely well built one. The `UPDATE` statement's `WHERE` clause repeats every precondition for "this listing is actually still buyable" (right listing, right domain name, the listing's own blockchain confirmation already succeeded, it is currently marked live and available), and only one caller among any number of concurrent buyers can possibly match that `WHERE` clause and flip `purchaseStatus` to `'In Progress'`, because the moment the first one succeeds, the row no longer satisfies `purchaseStatus: 'Available'` for anyone else. This is the same idea as a database row lock, achieved with a single statement instead of an explicit transaction, and it correctly prevents two people from both believing they bought the same domain.

What is not protected is the second half. The reservation and the `buyDomainRepo.save(newBuyDomain)` that follows it are two separate statements, not wrapped in a shared transaction. If the process crashes, or `buyDomainRepo.save` throws (a duplicate `blockchainTxHash`, since that column is `unique`, is a realistic way for this to happen if a retried request reuses the same transaction hash), the `catch` block logs the error and rethrows as a 500, but it never reverts the reservation it just made. The listing is left sitting at `purchaseStatus: 'In Progress'` forever, with no `BuyDomainListing` row to reconcile against it, meaning `transaction-cron`'s own reconciliation loop (which only ever looks at rows that already exist in `BuyDomainListing`) will never touch it either. This is precisely the "half completed purchase" failure category worth worrying about in a real payment system, a domain silently taken off the market with no purchase actually on file and no automatic path back to `Available`. It would need a human, or a dedicated diagnostic query, to notice and fix.

`BuyDomainListing` itself carries the same `reconciliationStatus` claim marker pattern as `DomainListing`, explained fully in `04`, plus `retryCount` (bumped, capped at seven attempts, by the cron whenever a blockchain receipt is not yet available) and a `unique` constraint on `blockchainTxHash`, which is what makes the crash scenario above a real, reachable one rather than a theoretical concern.

## Verifying the chain actually agrees, not just trusting the frontend's claim

`marketplace/abi/marketplace-event-decoder.ts` is worth reading directly, because it is the piece that keeps this whole on chain flow honest. Anyone could, in principle, call `POST /marketplace/buy-domain` with any `price` and `buyer` they like, that is just an HTTP body. What actually confirms a sale happened is `verifyNewSaleEvent`, run later by the cron against the real transaction receipt pulled straight off the blockchain node:

```ts
export function verifyNewSaleEvent(event, expected: ExpectedSale): SaleVerificationResult {
  if (!event || event.name !== 'NewSale') return { matches: false, reason: 'NewSale event not found in transaction logs' };
  if (event.args.listingId.toString() !== expected.listingId) return { matches: false, reason: 'listingId mismatch' };
  if (event.args.tokenId.toString() !== expected.tokenId) return { matches: false, reason: 'tokenId mismatch' };
  if (eventBuyer !== expected.buyerAddress.toLowerCase()) return { matches: false, reason: 'buyer mismatch' };
  if (!usdAmountsMatch(eventPrice, expected.priceInUSD)) return { matches: false, reason: 'price mismatch' };
  return { matches: true };
}
```

Every field the API request claimed, listing, token, buyer, price, gets checked against what the smart contract itself actually emitted on chain, and only a full match lets the cron mark the purchase `Success`. A mismatch on any single field, including a price that does not agree within one cent, marks the whole purchase `Failed` and releases the listing back to `Available`. This is what makes it safe for the initial `buyDomain` call to trust the request body at all, the request body only ever creates a `Pending` row, nothing about the transaction is treated as final until the chain itself has confirmed it, which is exactly the discipline a system moving real money over an inherently untrustworthy client needs.

## Frontend note

If you have built a UI for an NFT marketplace or any peer to peer resale flow before, this should feel familiar in shape, list an item, someone else buys it, both sides wait for a blockchain confirmation before anything is final. What is worth taking away as a backend lesson is the layered trust model, the API accepts a claim eagerly (so the UI can show "purchase pending" immediately) but nothing is marked final until an independent, un-spoofable source (the chain itself) has verified every detail of that claim, and the one place that discipline slips, the plain, un-transacted gap between reserving a listing and recording the buy, is exactly the kind of subtle bug this pattern is supposed to protect against elsewhere.
