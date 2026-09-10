# 05. The Reputation Score

## Opting in requires a real domain, not just a click

Reputation tracking is opt in, and the opt in itself has a real prerequisite, `ReputationService.optIn` refuses unless the user already has a `primaryDomainId` set, and that primary domain has to be a genuine `.og` domain the user actually owns, checked by re fetching it and confirming its TLD:

```ts
const [, tld] = await splitDomainNameAndTLD(domain.domainName);
if (tld !== 'og') throw new BadRequestException('Primary domain must be a .og domain');
```

Setting a new primary domain also resets the user's GM streak for that domain and clears any stale primary domain reference left behind if the domain changed owners, and opting in with a domain different from whatever domain a previous score was tied to purges that old score outright rather than letting it linger. The score itself is calculated immediately on opt in, wrapped in its own try catch so a scoring failure never rolls back a successful opt in, the user is told their score will simply be calculated on the next scheduled run instead.

## Six components add up to one number, capped at one thousand

`calculateTotalScore` in `ReputationService` runs six scoring calls in parallel and sums their results, each with its own maximum, `evmActivityScore` up to 250, `gmStreakScore` up to 200, `domainCountScore` up to 150, `domainTenureScore` up to 150, `nftActivityScore` up to 125, and `contractActivityScore` up to 125. Every normalizing formula lives in one pure, dependency free file, `scoring/score-normalizer.ts`, deliberately kept free of any database or API calls so the math itself is easy to test and reason about on its own.

The EVM activity score is the most involved. `EvmScoreService.calculate` queries Alchemy across five chains, Ethereum, Polygon, Arbitrum, BNB, and Base, for every wallet a user has connected, pulling transaction counts, first transaction date, ERC20 token count, and NFT count per chain, then combines those into one 0 to 1 score per wallet using fixed weights:

```ts
const WEIGHT_TX_COUNT = 0.40;
const WEIGHT_WALLET_AGE = 0.30;
const WEIGHT_ERC20 = 0.20;
const WEIGHT_NFT = 0.10;
```

When a user has more than one wallet, the scores are not simply averaged, the best wallet counts for seventy percent of the result and the average of the rest counts for the remaining thirty percent, so connecting a second, less active wallet can only ever help a score, never dilute it. Every one of the six calculators is defensive about failure independently, if a chain's data fetch fails, that chain is dropped and scoring continues on whatever chains succeeded, and if an entire component genuinely cannot be computed, the previous, already stored value for that component is preserved rather than the whole score collapsing to zero, each failure path is logged with a clear warning naming exactly which component fell back and why.

The GM streak score, domain count score, domain tenure score, and NFT and contract activity scores all reuse the loyalty mechanics covered elsewhere in this cluster, `GmScoreService` reads the exact same `GmStreakEntity` row described in the GM note, `DomainScoreService` counts a user's `.og` domains and separately measures how long the primary one has been held (falling back to a domain's own `createdDateTime` if a dedicated tenure record has not been populated yet), and the NFT and contract activity scores simply count how many of a user's own NFT collections and deployed contracts, from entirely separate feature components, have reached a `CONFIRMED` deployment status.

## The nightly cron, and the flag that freezes perk claims while it runs

`ReputationService.dailyCronRecalculate` runs at 3am every day and recalculates every user whose score is either older than the recalculation interval or who has been active in roughly the last week, in batches of one hundred, with a short delay between each user specifically to stay under Alchemy's rate limits, and up to three retries per user with exponential backoff before giving up on that one user for the run.

The part worth understanding closely is what happens around that batch, not inside it. Before the batch starts, a flag is set:

```ts
await this.systemFlagRepo.setCronRunning(true);
// ...batch runs...
await this.systemFlagRepo.setCronRunning(false); // in a finally block
```

That flag lives in its own tiny table, `tbl_system_flag`, and every perk claim attempt checks it first, refusing with a 503 if the cron is currently running. The comment explaining why is worth repeating directly, because it names a real fairness bug this code is specifically preventing, a user whose score gets recalculated early in the batch could otherwise claim a limited supply perk before a still queued user's score has even been touched yet, effectively cutting in line ahead of someone whose true, current score might have qualified them just as well. The flag also auto clears itself if it is ever left stuck for more than two hours, a defensive guard against the cron process crashing mid batch and leaving claims frozen indefinitely.

## The tier ladder, and what a user actually sees

Four tiers sit on top of the raw score, Bronze under 250, Silver from 250, Gold from 500, and Platinum from 750, and `getTier` is the one function every other read path calls to turn a number into a label consistently. `GET /reputation/score/me` returns the full breakdown per component along with its own max, the user's next tier and how many points away it is, how many wallets are connected out of a maximum of five slots, and a short list of plain language recommendations generated by comparing each component's score against its own maximum, "Check in daily to build your GM streak and earn more points" being the direct GM tie in. A score is also flagged `stale: true` in that response if it has not been recalculated in the last 48 hours, which is a real signal a frontend could use to explain why a number on screen looks out of date without the user having done anything wrong.

There is also a public side to all of this, `GET /reputation/score/:primaryDomainId` and `GET /reputation/leaderboard` both work without authentication, returning any opted in user's score by their domain, or a ranked list of everyone who has opted in, filterable by tier and by an all time or weekly window. Opting out is not a literal feature here, but the guard is effectively the same one that gates every other route, once `isReputationOptedIn` is false a lookup for that user's score returns 404 instead of leaking a stale number.
