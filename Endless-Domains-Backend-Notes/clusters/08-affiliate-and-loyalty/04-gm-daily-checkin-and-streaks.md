# 04. GM, the Daily Check In and Its Streak

## Confirming what "GM" actually means here

"GM" is not an abbreviation invented for this codebase, it is the real Web3 community greeting, short for "good morning", used as a daily ritual across crypto Twitter and Discord communities regardless of what time zone or actual hour someone posts it in. This codebase treats it exactly that way, and confirms it directly in a controller comment and a Swagger description:

```ts
// GmController
@Post('checkin')
@ApiOperation({
    summary: 'Daily GM Check-in',
    description: 'User performs daily "Say GM" check-in. One check-in allowed per primary domain per chain per calendar day...',
})
```

So the feature is a literal daily check in, gamified with streaks, exactly like a habit tracking app, except the "check in" is a real signed blockchain transaction rather than a button press, and it is only available to someone who already owns one of the company's `.og` domains, enforced by `OgDomainGuard` on every GM route, and who has separately opted into reputation tracking, enforced by `ReputationOptInGuard`.

## A check in has to be a real transaction

`CreateGmCheckinDto` asks the frontend for a wallet address, a chain name, and a `txHash`, and nothing else. The interesting work is entirely server side, inside `GmVerificationService.verifyOnChainTransaction`, which the class level comment lays out as seven ordered checks, chain resolution, transaction existence, transaction success, a GM event log existing in the receipt, the event's emitting contract address matching the expected one for that chain, the event's sender matching the wallet address the user claims, and the transaction being no older than ten minutes.

```ts
// src/components/reputation-gm-perk/gm/services/gm-verification.service.ts (lines 80 and 83, ethers v6 form since commit 71fd2fec)
const GM_EVENT_TOPIC = ethers.id('GM(address,string,uint256)');
const MAX_TX_AGE_MS = 10 * 60 * 1000;
```

The contract address check is deliberately validated against the emitting log's address, not against `tx.to`, and a comment explains exactly why, a sponsored gas or account abstraction transaction routes `tx.to` to an entry point or smart account rather than to the GM contract itself, so checking the log is the only way this stays correct for those wallets too. The event itself is decoded with a fixed ABI, `event GM(address indexed sender, string domainName, uint256 timestamp)`, and both the sender and, when a primary domain name is known, the domain string inside that event are compared against what the user submitted, so a transaction genuinely has to have been sent by that wallet, for that domain, to be accepted.

`GmService.checkin` runs this verification only after two cheap database checks pass first, whether the user already checked in today on this chain, and whether this exact `txHash` has ever been used before, both explicitly to avoid spending an RPC call on a request that would fail anyway.

```ts
const alreadyCheckedIn = await this.gmCheckinRepository.checkIfAlreadyCheckedInToday(userId, primaryDomainId, chainIdentifier);
if (alreadyCheckedIn) throw new ConflictException('You have already checked in today. Come back tomorrow!');
```

Uniqueness is per chain, not per day globally, a comment and the entity's own composite unique constraint both confirm the same user can GM on several different chains on the same calendar day, they just cannot GM twice on the same chain on the same day. `GmCheckinEntity` also has a plain `unique: true` on `txHash` by itself, so the same transaction hash can never be attached to two different check ins even across users, which is the actual database level defense against replaying someone else's transaction.

## The streak, and the races it has to survive

`GmStreakEntity` (`tbl_gm_streak`) keeps one row per user per primary domain, `currentStreak`, `longestStreak`, `lastCheckinDate`, and `totalCheckins`. Because a user can check in on more than one chain in a day, the service has to be careful that only the first check in of a given day advances the streak, everything after that on the same day is a no op as far as the streak is concerned.

```ts
const isFirstCheckinToday = !(await this.gmCheckinRepository.hasAnyCheckinOnDate(userId, primaryDomainId, today));
```

That flag is captured before the check in row is inserted, and the actual streak update is written as a single conditional SQL statement guarded by `lastCheckinDate IS DISTINCT FROM :guardDate`, specifically so that if two check ins on two different chains both land at almost the same moment and both believe they are the first of the day, only one of them actually increments the streak, the second one's update simply matches zero rows because the first one already moved `lastCheckinDate` to today.

Missing a day resets the streak, but not inside the check in flow itself, `ReputationService.handleUserLoginEvent` is what actually catches this, on every login it calls `checkAndResetStreakIfMissedDay`, which compares `lastCheckinDate` against both today and yesterday in UTC, using calendar day arithmetic rather than raw millisecond subtraction specifically so daylight saving transitions cannot corrupt the comparison, and zeroes `currentStreak` if neither matches. A user who does not log back in simply keeps whatever streak they last had until the next time they do, the reset is lazy, not scheduled.

## The founding member badge, and why it needs a database lock

Buried inside the same `checkin` flow is a one time, capped badge, the first five hundred users ever to GM get `isFoundingMember` set permanently on their `User` row. Because this runs on every single check in from potentially many users at once, a plain "count then insert" would let more than five hundred people slip through during a burst of concurrent requests. `assignFoundingMemberBadge` avoids that with a Postgres transaction level advisory lock taken before the count check even runs:

```ts
await queryRunner.query(`SELECT pg_advisory_xact_lock(${FOUNDING_MEMBER_LOCK_KEY})`);
const [{ count }] = await queryRunner.query(`SELECT COUNT(*) AS count FROM "tbl_user" WHERE "isFoundingMember" = true`);
```

The lock key, `42100`, is a plain integer with a comment warning it must never be reused by any other feature, since Postgres advisory locks are just numbers with no namespacing of their own, two unrelated features sharing the same lock key would end up serializing against each other for no reason. Failure here is deliberately isolated too, `checkin` wraps this whole call in a `try/catch` that swallows the error, because the check in and streak update have already been committed by this point, and a badge assignment failing should never look like the check in itself failed.

## Supported chains are a static list, not a database table

`SUPPORTED_CHAINS` inside `GmService` is a large hardcoded array covering roughly twenty chains, from Ethereum, Polygon, and Base down to newer or smaller networks like Monad, Soneium, and several testnets, each entry carrying a display name, logo URL, and a `verificationType`. Every chain in the list currently has `contractAddress: null` in this array, the real, secret contract addresses live in AWS Secrets Manager instead and are looked up separately inside `GmVerificationService` through `CHAIN_CONFIG`, keyed by the same lowercase chain name, so the public facing chain metadata and the actual verification configuration are intentionally kept apart, one is safe to return to any client through `GET /gm/supported-chains`, the other never leaves the backend.

## Update from the October 2026 uat pull

The GM feature did not change behaviour in this pull, but both of its services were migrated from ethers v5 to ethers v6 in commit `71fd2fec` ("Ether versoin 6", 11 September 2026), and both received their first spec files in the same commit. The `GM_EVENT_TOPIC` excerpt above has been corrected in place.

In `src/components/reputation-gm-perk/gm/services/gm-verification.service.ts` the changes are line 80 (`ethers.utils.id` to `ethers.id`), line 178 (`new ethers.providers.JsonRpcProvider(rpcUrl)` to `new ethers.JsonRpcProvider(rpcUrl)`), lines 181, 199 and 315 (the `TransactionResponse`, `TransactionReceipt` and `Block` types moving out of `ethers.providers`), and line 270 (`new ethers.utils.Interface([...])` to `new ethers.Interface([...])`). In `src/components/reputation-gm-perk/gm/services/gm.service.ts` the only change is line 330, `ethers.utils.getAddress(rawWalletAddress)` to `ethers.getAddress(rawWalletAddress)`. `ethers.id(text)` is `keccak256(toUtf8Bytes(text))`, which for an event signature string like `GM(address,string,uint256)` is exactly the `topics[0]` value every GM log carries, so the `receipt.logs.find(log => log.topics[0] === GM_EVENT_TOPIC)` at line 224 still matches byte for byte.

One v6 behaviour change happens to be handled correctly here without anyone needing to touch the code. In v5, `iface.parseLog(...)` threw for a log it could not match. In v6 it returns `null` instead. At line 274 the result goes straight into `decoded.args.sender`, so a `null` would throw `TypeError: Cannot read properties of null`, but that line sits inside the `try` at lines 269 to 287 (the `catch` starts at line 277), so the `catch` turns it into the same clean "Could not decode GM event log" rejection the v5 code produced. In practice it cannot even happen, because step 4 already chose the log by its GM topic.

Three honest risks remain, two of them new with v6 and one older.

First, line 178 builds a brand new `JsonRpcProvider` on every check in, with no chain id and no `{ staticNetwork: true }`. The marketplace v2 code explains, at `src/components/marketplacev2/order/order.service.ts` lines 154 to 159, that without that option an unreachable RPC makes a v6 provider retry network detection every second, indefinitely. For GM that means a dead `POL_GM_RPC` (or any other chain's key) can make a check in request stall instead of returning "Failed to fetch transaction from chain.", and leaves a retrying provider behind per attempt. Since `chainConfig.chainId` is already known at that point, passing it with `staticNetwork: true` is a one line fix.

Second, v6 types `provider.getBlock()` as `Promise<Block | null>`. The repository's `tsconfig.json` does not turn on `strictNullChecks`, so `let block: ethers.Block` at line 315 compiles anyway. If the node returns `null` for the block (a load balanced RPC whose second node is a block behind the first is the realistic case), nothing throws inside the `try` at lines 316 to 323, and `block.timestamp * 1000` at line 325 throws a `TypeError` that escapes the function. The user sees a 500 instead of a polite rejection, and because the check in has not been saved yet, they can retry, so the damage is a confusing error rather than corrupted data.

Third, an older ordering bug in `GmService.checkin`. Line 330 calls `ethers.getAddress(rawWalletAddress)` before line 335 checks `if (!rawWalletAddress || !chainIdentifier)`, and it is not inside a `try`. `CreateGmCheckinDto` only validates `walletAddress` with `@IsNotEmpty()` and `@IsString()`, so a string like `'hello'`, or a mixed case address whose EIP 55 checksum is wrong (which a wallet would never send but a hand written client could), makes `getAddress` throw an ethers `INVALID_ARGUMENT` error. That is not an `HttpException`, so NestJS answers with a generic 500 instead of a 400. Moving the normalisation into a `try` that throws `BadRequestException`, the way `contract-deployment.service.ts` lines 42 to 46 already do, would fix it.

The two new spec files are worth reading as examples of good unit tests in this codebase. `gm-verification.service.spec.ts` (14 tests) mocks only `ethers.JsonRpcProvider` through `jest.mock('ethers', ...)` with `jest.requireActual` keeping the rest of ethers real, and then builds genuine GM logs with `iface.encodeEventLog(fragment, values)` (lines 23 to 27), so the real v6 `Interface.parseLog` is exercised on real encoded bytes rather than a mocked decode. It has one test per rejection reason across all seven steps (unsupported chain, missing RPC, fetch failure, missing transaction, receipt failure, pending receipt, reverted status, missing GM event, wrong contract, wrong sender, wrong domain, stale block, block fetch failure) and one all steps pass test. `gm.service.spec.ts` (4 tests) shows a pattern specific to this service: `GmService` gets its repositories through property level `@Inject(...)` decorators (lines 295 to 308) rather than constructor parameters, so the spec constructs it with only the verification service and a fake connection, then assigns `(service as any).gmCheckinRepository` and friends by hand. The fake connection's `createQueryRunner` deliberately throws, which exercises the swallowed founding member failure path described above for free. One test pins `ethers.getAddress('0xd8da6bf26964af9d7eed9e03e53415d37aa96045')` to the independently known checksum `0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045`, and three use `jest.useFakeTimers()` with `jest.setSystemTime(...)` to prove the streak extends from yesterday, resets after a gap, and rolls correctly from 28 February to 1 March 2026. None of the tests cover the `getBlock` null case or the invalid wallet address case above. The codebase wide story is in [../12-marketplace-v2/12-ethers-v6-migration-and-the-new-test-suite.md](../12-marketplace-v2/12-ethers-v6-migration-and-the-new-test-suite.md).
