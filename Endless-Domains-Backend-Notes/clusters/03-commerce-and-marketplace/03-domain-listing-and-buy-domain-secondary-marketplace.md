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
// src/components/marketplace/abi/listing-id.util.ts (lines 7 to 17, native bigint since commit 71fd2fec)
export function assertNonZeroListingId(listingId: string): void {
  let value: bigint;
  try {
    value = BigInt(listingId);
  } catch {
    throw new BadRequestException('Invalid listingId');
  }
  if (value === 0n) {
    throw new BadRequestException('listingId 0 is not a valid listing');
  }
}
```

(Before the ethers v6 upgrade this used `BigNumber.from(listingId)` and `value.isZero()` from ethers v5; the update section at the end of this note explains the switch to plain JavaScript `BigInt`.) The comment above it explains why this exists, the marketplace contract's own `tokenToListing` mapping defaults any uninitialized entry to `0`, so a `listingId` of `0` can never legitimately refer to a real listing, only to a bug or a spoofed request, and this guard rejects it before a single database write happens. Second, duplicate active listings are blocked at the application layer, not the schema, `create` looks up any existing row for the same `domainName` with `status: 'Active'` and throws a different message depending on whether the caller already owns that active listing or someone else does, a small but real UX distinction a frontend error message should preserve.

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
// src/components/marketplace/abi/marketplace-event-decoder.ts (lines 63 to 84, condensed; `event` is typed ethers.LogDescription since the v6 upgrade)
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

## Update from the October 2026 uat pull

Two files this note quotes were rewritten in commit `71fd2fec` ("Ether versoin 6", 11 September 2026) as part of moving the whole backend from ethers v5 to ethers v6, and both excerpts above have been corrected in place. The listing and buying business logic did not change, and the reservation gap described above is still open. It is also worth knowing up front that this whole secondary marketplace now has a successor being built next to it, `src/components/marketplacev2`, based on the Seaport protocol rather than the custom `NFTDomainsMarketplaceV5` contract. The v1 code in this note is still registered and still running, but new marketplace work is happening in v2, described starting at [../12-marketplace-v2/01-what-marketplace-v2-is-and-the-module-map.md](../12-marketplace-v2/01-what-marketplace-v2-is-and-the-module-map.md), with its order model in [../12-marketplace-v2/03-seaport-primer-and-the-order-entity.md](../12-marketplace-v2/03-seaport-primer-and-the-order-entity.md).

### `assertNonZeroListingId` no longer uses an ethers type at all

The old version imported `BigNumber` from `ethers` and called `BigNumber.from(listingId)` and `value.isZero()`. ethers v6 removed `BigNumber` entirely in favour of the native JavaScript `bigint`, so the new version at `src/components/marketplace/abi/listing-id.util.ts` lines 7 to 17 drops the ethers import and uses `BigInt(listingId)` and `value === 0n` instead. Two reasons this is the right shape. First, a listing id is a `uint256` on chain and can be far larger than `Number.MAX_SAFE_INTEGER` (9007199254740991), and `bigint` represents any integer exactly, while `Number('99999999999999999999')` silently rounds. Second, `BigInt(...)` throws a `SyntaxError` for anything that is not an integer string (`'abc'`, `'1.5'`), which the `try`/`catch` turns into a clean `400 Invalid listingId`.

There is one small behavioural difference worth knowing. `BigInt('')` and `BigInt('   ')` do not throw, they return `0n`, whereas v5's `BigNumber.from('')` threw "invalid BigNumber string". So an empty `listingId` used to fail with "Invalid listingId" and now fails with "listingId 0 is not a valid listing". It is still a 400 and still rejected before any database write, so nothing unsafe gets through, but a frontend that matched on the exact message text would see a different string. `BigInt` also accepts `0b` and `0o` prefixed strings, which `BigNumber.from` did not, a harmless widening. The existing spec `listing-id.util.spec.ts` (which already carried Sprint 04 test case U4.6) gained a fifth test asserting that `'99999999999999999999'` does not throw, documenting exactly why `BigInt` rather than `Number` was chosen.

### The event decoder and the `parseLog` null trap

`src/components/marketplace/abi/marketplace-event-decoder.ts` had every type renamed (`ethers.utils.Interface` to `ethers.Interface` at line 4, `ethers.utils.LogDescription` to `ethers.LogDescription` at lines 15, 16, 36, 64 and 94), but the change that actually matters is this one:

```ts
// src/components/marketplace/abi/marketplace-event-decoder.ts (lines 15 to 31)
export function decodeMarketplaceLogs(logs: RawLog[]): ethers.LogDescription[] {
  const decoded: ethers.LogDescription[] = [];
  for (const log of logs || []) {
    try {
      // ethers v6's Interface.parseLog returns null for a non-matching topic instead of
      // throwing (v5 threw here) — must be filtered explicitly, or a null slips into the
      // decoded array and callers like findMarketplaceEvent would throw when reading .name.
      const parsed = marketplaceInterface.parseLog(log);
      if (parsed) {
        decoded.push(parsed);
      }
    } catch {
      continue;
    }
  }
  return decoded;
}
```

A real transaction receipt for a marketplace sale contains logs from several contracts, the marketplace itself plus the NFT contract's `Transfer` and possibly token contracts. In v5, `parseLog` on a foreign log threw, the `catch` skipped it, and only marketplace events were collected. In v6 the same call quietly returns `null` instead. Without the new `if (parsed)` guard, every foreign log would have pushed a `null` into `decoded`, and `findMarketplaceEvent`'s `.find((event) => event.name === eventName)` at line 37 would then crash with `TypeError: Cannot read properties of null (reading 'name')` on the first foreign log it reached, which in `transaction-cron/cron.service.ts` line 403 would have stopped every buy confirmation dead. This is the single most important behavioural catch in the whole migration for this cluster. The `catch` is still needed, because v6 still throws for a log whose topic matches but whose data cannot be decoded.

`normalizeUsdPrice` at lines 41 to 43 now takes a `bigint` instead of a `BigNumber` and calls `ethers.formatUnits(priceInUSD, USD_PRICE_DECIMALS)`, the v6 spelling of `ethers.utils.formatUnits`. It still returns a plain JavaScript number through `parseFloat`, so `usdAmountsMatch` keeps comparing two numbers with a one cent tolerance, and nothing `bigint` ever reaches a JSON response from this file. The comparisons inside `verifyNewSaleEvent` and `verifyListingAddedEvent` use `event.args.listingId.toString()` and `event.args.tokenId.toString()` against the stored strings, and `bigint.toString()` gives the same decimal string `BigNumber.toString()` did, so those checks are unchanged in behaviour. Notice that the receipts fed into this decoder still come from web3.js (`web3.eth.getTransactionReceipt` at `cron.service.ts` line 398), so this is a case of ethers v6 decoding logs that web3 v1 fetched, which works because both use the same plain `{ topics, data }` log shape.

The spec `marketplace-event-decoder.spec.ts` (15 tests, covering Sprint 04 test cases U4.1 to U4.7) was rewritten to build its fixture logs with v6: `ethers.BigNumber.from('42')` became the literal `42n`, `ethers.utils.parseUnits('123.45', 6)` became `ethers.parseUnits('123.45', 6)`, and `ethers.utils.keccak256(ethers.utils.toUtf8Bytes(...))` became `ethers.keccak256(ethers.toUtf8Bytes(...))`. Its existing test "ignores logs that are not one of the known marketplace events" now doubles as the regression test for the `parseLog` null change above, since a random topic now goes through the `null` branch instead of the `catch`. A related v6 fix outside this cluster, `findByTokenId` on the domain detail repository gaining a `registryAddress` scope, is used only by marketplace v2 and is covered in [../04-domain-core-product/01-domain-entities-what-a-domain-actually-is.md](../04-domain-core-product/01-domain-entities-what-a-domain-actually-is.md). The codebase wide migration story is in [../12-marketplace-v2/12-ethers-v6-migration-and-the-new-test-suite.md](../12-marketplace-v2/12-ethers-v6-migration-and-the-new-test-suite.md).
