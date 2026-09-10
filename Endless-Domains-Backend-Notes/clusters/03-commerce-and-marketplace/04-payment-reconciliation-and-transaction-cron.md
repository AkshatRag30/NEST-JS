# 04. Transaction Cron, the Job That Reconciles What the Blockchain Actually Did

## Why this exists

Every write covered in `03`, listing a domain or buying one, starts life as a row marked `blockchainStatus: 'Pending'`, created the instant a frontend hands the backend a transaction hash, before anyone actually knows whether that transaction will succeed, fail, or simply never confirm. A card payment either succeeds or fails within seconds, in the same request. A blockchain transaction does not work that way, it sits in a mempool for an unpredictable amount of time, and there is no webhook a blockchain sends you the moment it lands, you have to go and ask. `CronService`, running once a minute (`@Cron(CronExpression.EVERY_MINUTE)`), is the piece of this system whose entire job is going and asking, for every pending listing and every pending purchase, across five different blockchains (`web3bnb`, `web3eth`, `web3poly`, `web3arb`, `web3base`, one `Web3` instance per chain, each pointed at its own RPC provider URL and marketplace contract address from config).

This is genuinely advanced, real world payment infrastructure, the same underlying problem every crypto payment system and increasingly many card systems (anything using asynchronous capture, buy now pay later, or bank transfers) eventually has to solve, and it is worth reading slowly rather than skimming.

## The claim pattern, an alternative to a database transaction

Both reconciliation loops (`checkBlockchainTransactions` for listings, `checkBuyDomainTransactions` for purchases) start the same way, and the shared helper is worth reading on its own:

```ts
private async claimForReconciliation(repo: Repository<ReconciliableEntity>, ids: string[]): Promise<Set<string>> {
  const result = await repo.createQueryBuilder()
    .update()
    .set({ reconciliationStatus: 'Reconciling' })
    .where('id IN (:...ids)', { ids })
    .andWhere('"reconciliationStatus" = :pending', { pending: 'Pending' })
    .returning(['id'])
    .execute();
  return new Set((result.raw as Array<{ id: string }> ?? []).map((row) => row.id));
}
```

The cron pulls a batch of candidate rows first, then tries to claim all of them in one `UPDATE ... WHERE reconciliationStatus = 'Pending' RETURNING id`. Because that `WHERE` clause requires the row to still be `Pending` at the moment the statement runs, and because a single `UPDATE` statement is inherently atomic per row, this is exactly the same compare and swap trick used in `buy-domain`'s purchase reservation, just applied to "which cron tick owns this row" instead of "which buyer owns this listing." The comment above it spells out exactly why this matters, this job runs every single minute, and a slow tick (a slow RPC provider, a burst of pending transactions) can easily still be running when the next tick starts, without this claim, two overlapping ticks could both pick up the same pending row and process it twice, sending duplicate emails or double counting a sale. The `RETURNING` clause is what lets the loop know which rows it actually won the claim on versus which ones a slower, still running previous tick already grabbed, `if (!claimedIds.has(listing.id)) continue;` skips anything it lost the race for.

`releaseReconciliationClaim`, run in a `finally` block no matter which branch of the per row logic executed or whether it threw, decides where the row goes next: back to `'Pending'` if the underlying `blockchainStatus` is still `'Pending'` itself (so the next tick retries it), or to `'Done'` if it reached a terminal state (`Success` or `Failed`), so a row is never picked up and reprocessed once its outcome is actually settled.

## Confirming a listing actually went live

`checkBlockchainTransactions` asks the right chain's node for the transaction receipt, and works through a deliberately ordered set of failure checks before it will ever call a pending listing successful: no receipt yet increments `retryCount` and gives up (marks `Failed`) only after seven consecutive misses; a receipt whose `to` address does not match the expected marketplace contract address is an immediate `Failed`, regardless of receipt status, someone's transaction hash pointed somewhere else entirely; a receipt with `!receipt.status` (an on chain revert) is `Failed`. Only once all of that passes does it walk `receipt.logs`, decode them against the known marketplace ABI, and look specifically for a `ListingAdded` or `ListingRemoved` event that matches this row's own `listingId` and `blockchainTxHash`.

Confirming `ListingAdded` runs one more real check most systems built by less careful engineers would skip, `verifyListingAddedEvent` (covered in `03`) compares the event's `tokenOwner` against the wallet address this codebase has on file for the user who submitted the listing. A transaction landing successfully and emitting the right event is not, on its own, proof it came from the right person, this closes that gap.

## Confirming a purchase actually happened, and the seller actually gets paid

`checkBuyDomainTransactions` does the equivalent work for the buy side, and it is worth reading the exact sequence of what happens on a confirmed, verified sale, because it is a small real world example of a multi step business process being driven off one blockchain confirmation:

```ts
const resp = await this.buyDomainRepo.update({ blockchainTxHash: receipt.transactionHash, listingId: listing.listingId }, { blockchainStatus: 'Success', updatedTime: currentTimeInUnix, pricePerToken: eventPriceInUSD.toString() });
if (resp) {
    await this.domainListingRepo.update({ listingId: listing.listingId }, { ownerAddress: eventBuyer, purchaseStatus: 'Sold', status: 'Delisted', listingStatus: false });
    await this.domainListingservice.sendEmailToBuyerAndSeller(listing.listingId);
    const walletDetails = await this.wletRepoInterface.findByWalletWithoutNetwork(user.ownerAddress);
    await this.fetchDomainService.updateDomainDetialTable({ userId: walletDetails.userId });
    await this.domainListingservice.getEmailNotificationWatchlist({ domainId: listing.domainId });
}
```

Marking the purchase successful, flipping the listing to sold and delisted, emailing both buyer and seller (the seller's email even computes their actual payout after the marketplace's cut, `soldAmount - soldAmount * 2.5 / 100`, right there in `DomainListingService.sendEmailToSeller`), refreshing the buyer's own domain inventory table, and notifying everyone who had this domain on their watchlist that it just sold, are five separate steps, none of them wrapped in a shared database transaction. In practice a crash mid sequence here is less dangerous than the one flagged in `03`, because every later step is either idempotent or just a notification, but it is still worth naming as the same underlying pattern repeated, this codebase generally chooses sequential writes plus good logging over explicit transactions for these multi step business flows, and gets away with it mostly because each individual step is designed to be safely re-runnable.

Every failure branch, no `NewSale` event found, the verification check failing, the wrong contract address, a plain revert, does the same two things together: mark the `BuyDomainListing` row `Failed`, and explicitly hand the listing back with `purchaseStatus: 'Available'` so it goes back on sale rather than staying stuck at `'In Progress'` forever. That release-back-to-Available step is the exact repair this cron is able to make for failures it actually gets to see, which only sharpens why the un-transacted gap flagged in `03` matters, that gap produces a stuck row this cron never even looks at, because it only iterates rows that already exist in `BuyDomainListing`.

## The smaller jobs riding along in the same file

`markExpiredDomains`, also every minute, flips any listing past its `endTime` to `status: 'Expired'`, and separately walks `WebhookPaymentInvoice` rows whose promotion `expiry_date` has passed, turning `isPromoted` back off on the underlying listing and flagging the invoice `promotion_expired_handled` so it is never processed twice. `updateTrendingScores`, hourly, recomputes marketplace trending scores by calling into the analytics module, covered in `09`.

## Frontend note

Every "your listing is pending confirmation" or "your purchase is processing" state a marketplace UI shows is this cron in disguise, the frontend cannot know the real outcome any faster than this job's next run, once a minute, discovers it, and every one of those pending states can resolve to a specific, named failure (contract mismatch, event verification failure, seven retries with no receipt) rather than a generic timeout, which is worth surfacing distinctly in the UI rather than flattening into one spinner that eventually just gives up.
