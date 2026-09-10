# 09. The Smaller Marketplace Features, and What All of This Looks Like From a Frontend Seat

## Watchlist, a toggle that quietly does two jobs

`Watchlist` (`tbl_watchlist`) is about as simple as an entity gets, `user_id`, `domain_id`, and a boolean `status`. What is worth noticing is `WatchlistRepository.insertWatchlist`, which treats "add to watchlist" and "remove from watchlist" as the same endpoint:

```ts
const existingEntry = await this.watchlistRepo.findOne({ where: { user_id, domain_id } });
if (existingEntry) {
    const newStatus = !existingEntry.status;
    await this.watchlistRepo.update({ id: existingEntry.id }, { status: newStatus });
    return { status: newStatus, message: newStatus ? 'Item saved to watchlist.' : 'Item removed from watchlist.' };
}
```

One `POST /marketplace/watchlist-add` call is a genuine toggle, the row never gets deleted, it just flips `status` back and forth, which is why `getWatchlist` and `getUserDomainWatchlistData` both filter `status: true` rather than simply returning every row for a user. This same table quietly feeds two other pieces of this cluster, the "recent sales" data in `buy-domain.service.ts` batch counts how many active watchers a sold domain had, and the domain listing service's own delisting logic emails every active watcher when a domain they were watching gets sold or expires.

## Premium and promoted domains, one paid, one free

These two are easy to conflate but are unrelated features. `promoted-domains` is a plain, unauthenticated, cached read (`promotedDomainDetailList`, `@Header('Cache-Control', 'public, max-age=60')` on its controller route) over listings with `isPromoted: true`, the flag that only ever gets set by the paid promotion flow covered in `08`. `premium-domains` (`PremiumDomainService`) is a different, free feature entirely, it calls an external domain appraisal API to estimate a domain's value, then applies its own small heuristic on top of that estimate:

```ts
public async analyzeDomain(domain: string) {
    const estimateVal = await this.appraiseDomain(domain);
    const category = this.isCategorizedTLD(domain); // is it a recognized web3 TLD at all
    const isThreeCharacter = this.isOneToThreeCharacterDomain(domain);
    const highValue = this.isHighValueDomain(estimateVal ?? 0); // > $1000
    let isPremium = category && (isThreeCharacter || highValue);
    return { domain, category, isThreeCharacter, highValue, isPremium, success: true };
}
```

A domain only ever gets flagged premium here if it is on a recognized web3 TLD list at all, and then only if it is also very short or independently appraised above a thousand dollars. This is analysis only, nothing here writes `isPremium` back onto a `DomainListing` row, that write actually happens through the unrelated `analytics.addTagsDomainListingData` endpoint, called separately by an admin, which is a small but real seam worth knowing, "is this domain premium" gets computed in one module and persisted through a completely different one.

There is a second, entirely separate domain appraisal implementation, `appraisel-domains` (note the folder's own spelling), which does not call any external API at all, it is a self contained, static scoring function built from a hardcoded list of premium web3 TLDs and per TLD dollar values, plus a small point system for name length, hyphens, and digits. Two different domain valuation systems living side by side in the same cluster, one calling out to a real appraisal service, one entirely local and rule based, is worth flagging exactly the way the earlier notes flagged the two separate Cryptomus integrations, always check which one a given caller is actually using before assuming "domain appraisal" means one specific thing in this codebase.

## Keyword suggestions, analytics, history, contact us, and redirect

`keyword` proxies the free Datamuse word association API for domain name suggestions, filtering out a configured block list of unwanted words, and falls back to a small hardcoded word list (`test`, `sample`, `demo`, `example`, `mock`) rather than ever failing outright if Datamuse itself is unreachable, a deliberate choice to keep a search-as-you-type feature always returning something rather than erroring.

`analytics` tracks per listing views, clicks, saves, and unsaves (`AnalyticsListing`), and separately computes a "trending" score every hour through `transaction-cron`'s `updateTrendingScores` job, using a single bulk SQL `UPDATE` with window functions (`MAX(...) OVER ()`, `MIN(...) OVER ()`) to normalize every listing's raw counts against the current min and max across the whole table in one pass, then decays that score exponentially by how long ago the listing's `startTime` was, `EXP(-0.05 * days_elapsed)`, so an old listing with a lot of historic clicks does not permanently outrank a genuinely hot new one. `history` is a read only, paginated feed that merges five completely different tables (buys, listings, delistings, promotions, and sales, plus a separate bulk listing aggregation using `ARRAY_AGG` and `jsonb_build_object`) into one unified activity feed for a user, sorted and paginated in application code after all five queries return, worth noting as a real example of "denormalize at read time across many tables" rather than maintaining one dedicated activity log table. `contact-us` and `redirect` are both small and self contained, a plain contact form submission, and a cross domain single sign on helper that mints a fresh access and refresh token pair and sets them as cookies scoped to `.endlessdomains.io` so a login on the main site carries over onto the separate marketplace subdomain.

## What all of this looks like from the other side of the API

If you have built a cart, a checkout flow, or a marketplace UI before, calling someone else's backend, this cluster is a good picture of what was actually happening underneath those calls the whole time.

Adding something to a cart is never just "store this item," it is a live price and availability lookup against a real, possibly external, possibly on chain source of truth, every single time, which is why the same add to cart call can succeed one moment and come back with "no longer available" the next, and why a frontend has to treat that response as a normal, expected outcome rather than a bug to work around.

A checkout submission and a blockchain purchase submission look similar from the UI's side, a spinner, then success or failure, but underneath, a card payment resolves synchronously in the same request while a blockchain purchase only ever starts as `Pending` and gets confirmed later by a background job running on a fixed schedule, `04`'s reconciliation cron, which is exactly why marketplace UIs so often show a distinct "pending confirmation" state that a plain checkout does not need.

A coupon box is quietly revalidating an entire cart, not just a code string, every time it is submitted, and a promotion payment (`08`) is a genuinely complete, if small, real payment integration, invoice creation, signature verified webhooks, idempotency guards against duplicate delivery, and over or under payment handling, hiding behind what looks to a frontend developer like one simple "boost this listing" button.

And the single most useful thing to take from this entire cluster if you are trying to grow from frontend into fullstack work is `05`'s diagnostics tool and the recurring "half completed state" callouts throughout `01` through `08`. Every "something went wrong, please try again" error message your own frontend code has ever shown a user was, on the other side of that API call, one of a genuinely small, nameable set of failure categories, a webhook that never arrived, a payment that succeeded but a downstream step that did not, two writes that should have happened together but were not wrapped in anything guaranteeing they would. Learning to ask "which one was it" instead of just retrying is most of what separates a frontend developer who consumes an API from one who can actually be trusted to build the other side of it.
