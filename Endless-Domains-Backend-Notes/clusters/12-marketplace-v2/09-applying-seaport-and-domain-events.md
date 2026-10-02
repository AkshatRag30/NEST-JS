# 09. Applying Seaport and Domain Events

## Where this picks up

File `08` explained the loop: every fifteen seconds the poller reads a range of confirmed Polygon blocks, fetches every log from the Seaport contract (`OrderFulfilled`, `OrderCancelled`, `CounterIncremented`) and every log from the Unstoppable Domains NFT contract on Polygon (`Transfer`, `ApprovalForAll`), and hands each one, Seaport first, to one of two application services. This file is about those two services, `src/components/marketplacev2/poller/application/seaport-event-application.service.ts` and `src/components/marketplacev2/poller/application/domain-event-application.service.ts`. For each of the five events it explains exactly what is decoded, which rows are read, which rows are written in which order and inside which transaction, how a replay of the same log is made harmless, and what can still go wrong. It finishes with the ordering guarantees the design relies on, how the poller interacts with the other writers of the same tables, and a walk through every spec file in the folder.

These are the only five events the poller fetches or decodes. There is no `OrdersMatched`, no ERC721 single token `Approval`, no ERC20 `Transfer` of USDT; if it is not in `abi/poller-events.abi.ts` the poller never sees it.

## The tables the poller writes, and the order state machine

The poller writes to four tables, none of which it owns except the cursor.

| Table | Entity | What the poller does to it | Idempotency device |
|---|---|---|---|
| `tbl_marketplacev2_orders` | `OrderEntity` (`order/entity/order.entity.ts`) | Conditional `UPDATE` of `status`, `fillTxHash`, `filledAt`, `filledBy`, `invalidReason` | Every `UPDATE` has a `status` condition in its `WHERE` |
| `tbl_marketplacev2_listing_status` | `ListingStatusEntity` | Upsert the seller's row to `UNLISTED` | `ON CONFLICT ("userId","domainName","tokenId") DO UPDATE`, naturally idempotent |
| `tbl_marketplacev2_listing_history` | `ListingHistoryEntity` | Insert a `SOLD`, `CANCELLED` or `INVALIDATED` row | `UNIQUE ("orderHash","eventType")` plus `INSERT ... ON CONFLICT DO NOTHING` |
| `tbl_marketplacev2_transactions` | `TransactionEntity` | Insert one `SALE` row per fill | `UNIQUE ("orderHash","transactionType")` plus `ON CONFLICT DO NOTHING` |

The order statuses come from `order/enum/order-status.enum.ts`: `active`, `cancelled`, `filled`, `expired`, `invalid`. `invalid` is always paired with an `invalidReason` from `order/enum/order-invalid-reason.enum.ts`, either `owner_changed` or `approval_revoked`. Here is who moves an order between them across the whole of marketplace v2, which matters because the poller has to coexist with the other writers.

| Transition | Writer | Trigger |
|---|---|---|
| (new) to `active` | `OrderService.create` | Seller submits a signed order |
| `active` to `cancelled` | `OrderService.cancel` (off chain) or poller `applyOrderCancelled` (on chain) | Seller clicks cancel in our UI, or calls Seaport `cancel` directly |
| `active` to `expired` | `OrderService.expireIfStale`, run by `ListingExpiryScheduler` every five minutes and on relist | `endTime` has passed |
| `active` to `invalid` (`owner_changed`) | Poller `applyTransfer` | Token moved to a wallet other than the maker |
| `active` to `invalid` (`approval_revoked`) | Poller `applyApprovalForAll` | Maker revoked Seaport's operator approval |
| `active`, `cancelled`, `invalid` or `expired` to `filled` | Poller `applyOrderFulfilled` | Seaport emitted `OrderFulfilled` for this `orderHash` and it verified |

`filled` is the only terminal state. That single rule, "the chain wins, and a fill beats everything", is what makes most of the ordering questions later in this file turn out to be safe.

## The shared toolkit both services use

### One transaction per log, with the side tables inside it

Every state changing method follows the same shape: do any network I/O first, then open `orderRepository.manager.transaction(async (manager) => { ... })`, run a conditional `UPDATE` on the orders table, and only if it affected a row, write the listing status, listing history and (for fills) transaction rows through the same `manager`. `ListingStatusService`, `ListingHistoryService` and `TransactionService` all take an `EntityManager` parameter for exactly this reason, which their doc comments spell out: the derived rows must commit or roll back with the order write that caused them.

### Why the inserts use `ON CONFLICT DO NOTHING` instead of try and catch

```ts
// src/components/marketplacev2/listing-status/listing-history.service.ts
async record(manager: EntityManager, event: ListingHistoryEvent): Promise<void> {
    await manager.createQueryBuilder().insert().into(ListingHistoryEntity).values({ ...event, createdAt: new Date() }).orIgnore().execute();
}
```

```ts
// src/components/marketplacev2/transaction/transaction.service.ts
async record(manager: EntityManager, event: RecordTransactionEvent): Promise<void> {
    await manager.createQueryBuilder().insert().into(TransactionEntity).values({ ...event, createdAt: new Date() }).orIgnore().execute();
}
```

This is a genuinely important Postgres lesson and the comment above `ListingHistoryService.record` explains it well. Inside a Postgres transaction, any statement that errors (including a unique violation) puts the whole transaction into an aborted state. If you catch the exception in JavaScript and carry on, every later statement fails with "current transaction is aborted", and if you reach `COMMIT`, Postgres silently performs a `ROLLBACK`. So "try to insert, catch the duplicate, ignore it" is not safe inside a transaction. `.orIgnore()` compiles to `INSERT ... ON CONFLICT DO NOTHING`, which never raises, keeps the transaction healthy, and makes a repeat insert a true no op at the database.

### Listing status upsert

```ts
// src/components/marketplacev2/listing-status/listing-status.service.ts
private async upsert(manager: EntityManager, userId: string, domainName: string, tokenId: string, status: ListingStatus): Promise<void> {
    await manager.getRepository(ListingStatusEntity).upsert({ userId, domainName, tokenId, status, updatedAt: new Date() }, ['userId', 'domainName', 'tokenId']);
}
```

TypeORM 0.3's `upsert` compiles to `INSERT ... ON CONFLICT ("userId","domainName","tokenId") DO UPDATE SET ...`. Writing `UNLISTED` twice is the same as writing it once, so this is idempotent by nature. The entity's comment is clear that this table is a cache for "my domains", never a source of truth; browse and validity always read the orders table.

### Resolving the seller's internal user

```ts
// src/components/marketplacev2/poller/application/seaport-event-application.service.ts
private async resolveOrderUserId(order: { maker: string; userId?: string | null }, context: string): Promise<string | null> {
    if (order.userId) {
        return order.userId;
    }
    const wallet = await this.walletRepo.findByWalletWithoutNetwork(order.maker);
    if (!wallet) {
        this.customLoggerService.warn(`${context} - no internal user found for wallet ${order.maker}, skipping listing-status/history update`);
        return null;
    }
    return wallet.userId;
}
```

The listing status and history tables are keyed by internal `userId`, while the chain only knows wallet addresses. Commit `96f221ec` added a `userId` column to `OrderEntity`, captured at create time from the authenticated request, after the team found that the older approach, an exact, case sensitive wallet lookup on every event, silently skipped the side table writes whenever the on chain address casing did not match the stored one. The fallback lookup now only runs for rows created before that column existed. When no user is found, the order write still happens and only the side tables are skipped, with a warning, so a missing mapping can never roll back the authoritative write.

Two small things to notice. The fallback runs inside the transaction callback but through `walletRepo`, which uses its own repository rather than the transaction's `manager`, so for legacy rows each event briefly holds two pool connections at once. And the identical method is copy pasted into both services and into `OrderService`.

### Decoding

```ts
// src/components/marketplacev2/poller/application/seaport-event-application.service.ts
private decode(log: DecodableLog, eventName: string): ReturnType<Interface['parseLog']> {
    try {
        const parsed = SEAPORT_INTERFACE.parseLog({ topics: log.topics as string[], data: log.data });
        if (!parsed || parsed.name !== eventName) return null;
        return parsed;
    } catch (err) {
        this.customLoggerService.error(`failed to decode expected "${eventName}" log: ${(err as Error).message}`);
        return null;
    }
}
```

In ethers version 6, `Interface.parseLog` returns `null` when `topics[0]` matches no event in the interface (version 5 threw), and throws when the data does not decode against the matched fragment. Either way this method returns `null`, and every caller treats `null` as "not ours, return false". The `parsed.name !== eventName` check is a belt and braces guard in case the dispatcher ever routes the wrong topic here. Decoded values follow version 6 conventions: `uint256` and `uint8` arrive as native `bigint`, addresses arrive already in EIP 55 checksum casing, and tuple arrays arrive as `Result` objects you can index and read by name.

## OrderFulfilled, a sale

### What the event carries

`OrderFulfilled(bytes32 orderHash, address indexed offerer, address indexed zone, address recipient, SpentItem[] offer, ReceivedItem[] consideration)`. Only `offerer` and `zone` are indexed. The `orderHash` is the EIP 712 hash of the signed order, which the backend computed server side when the order was created and stored as the primary key of `tbl_marketplacev2_orders`. That hash is the join key between the chain and the database. `offer` is what the seller gave up (the domain NFT), `consideration` is everything paid out (seller's USDT, fee USDT, and any extra items the fulfiller added), and `recipient` is who received the offer items, which the code treats as the buyer.

### Step by step

```ts
// src/components/marketplacev2/poller/application/seaport-event-application.service.ts
async applyOrderFulfilled(log: DecodableLog, mismatchedFillTxHashes?: Set<string>): Promise<boolean> {
    const decoded = this.decode(log, 'OrderFulfilled');
    if (!decoded) return false;

    const orderHash = decoded.args.orderHash as string;
    const order = await this.orderRepository.findOne({ where: { orderHash } });
    if (!order) {
        this.customLoggerService.debug(`OrderFulfilled for ${orderHash} - not one of ours, ignoring`);
        return false;
    }

    if (order.status === OrderStatus.FILLED) {
        this.customLoggerService.warn(`OrderFulfilled for ${orderHash} which is already 'filled' - terminal state, ignoring (idempotent replay or duplicate event)`);
        return true;
    }
```

1. Decode. On failure, return `false`.
2. Look up the order by `orderHash`. On Polygon, Seaport is shared by every marketplace that uses it, so the overwhelming majority of fills are somebody else's and end here, at debug level, returning `false`. This lookup is a primary key hit, so it is cheap even at high volume. It does rely on the stored hash being lowercase hex, which matches what ethers produces on both sides.
3. If the order is already `filled`, this is a replay (a restart, an aborted tick, a second poller). Return `true` without touching anything. This early exit is an optimisation; the conditional update below would also be a no op.

```ts
// src/components/marketplacev2/poller/application/seaport-event-application.service.ts
    const offer = decoded.args.offer as Array<{ itemType: bigint; token: string; identifier: bigint; amount: bigint }>;
    const consideration = decoded.args.consideration as Array<{ itemType: bigint; token: string; identifier: bigint; amount: bigint; recipient: string }>;

    const mismatch = this.verifyAgainstStored(order, offer, consideration);
    if (mismatch) {
        mismatchedFillTxHashes?.add(log.transactionHash);
        this.customLoggerService.error(`OrderFulfilled verification mismatch for ${orderHash} - NOT recording the fill, leaving the row for a human to look at. ${mismatch}`);
        return true;
    }
```

4. Verify the decoded event against what was stored. The method comment frames it well: "A matching orderHash is strong evidence, not proof". The checks run in a fixed order and the first failure wins.

```ts
// src/components/marketplacev2/poller/application/seaport-event-application.service.ts
private verifyAgainstStored(order: OrderEntity, offer: Array<{ identifier: bigint }>, consideration: Array<{ amount: bigint; recipient: string }>): string | null {
    if (offer.length !== 1) {
        return `expected offer.length 1, got ${offer.length}`;
    }
    if (consideration.length < 2) {
        return `expected at least 2 consideration items, got ${consideration.length}`;
    }
    const [sellerLeg, feeLeg] = consideration;
    if (sellerLeg.amount.toString() !== order.sellerUsdt) { /* seller leg amount mismatch */ }
    if (getAddress(sellerLeg.recipient) !== getAddress(order.maker)) { /* seller leg recipient mismatch */ }
    if (feeLeg.amount.toString() !== order.feeUsdt) { /* fee leg amount mismatch */ }
    if (getAddress(feeLeg.recipient) !== getAddress(this.chainConfig.feeRecipient)) { /* fee leg recipient mismatch */ }
    if (offer[0].identifier.toString() !== order.tokenId) { /* offer identifier mismatch */ }
    return null;
}
```

| Check | Compares | Why |
|---|---|---|
| Offer length is exactly 1 | `offer.length` | Our orders always offer exactly one NFT |
| At least two consideration items | `consideration.length` | Seller leg and fee leg. Relaxed from "exactly two" in commit `96f221ec` because Seaport lets a fulfiller append extra items (tips, aggregator fees) at fill time |
| Seller leg amount | `consideration[0].amount` vs `order.sellerUsdt` | Integer USDT minor units, compared as decimal strings, so no float or `Number` precision loss |
| Seller leg recipient | `consideration[0].recipient` vs `order.maker` | Checksum normalised on both sides |
| Fee leg amount | `consideration[1].amount` vs `order.feeUsdt` | |
| Fee leg recipient | `consideration[1].recipient` vs `chainConfig.feeRecipient` | Uses the current config, not the value at signing time |
| Token identifier | `offer[0].identifier` vs `order.tokenId` | `tokenId` exceeds `Number.MAX_SAFE_INTEGER`, which is why both sides are strings |

Not checked are the offer item's `token` address, the consideration items' `token` (USDT) and `itemType`. That is defensible, because Seaport only emits `OrderFulfilled` with a given `orderHash` if the executed items match the signed components, and the hash covers every token address. A residual edge: if `FEE_RECIPIENT` is ever rotated in the secret, every still active order signed with the old fee recipient will fail verification when it sells.

On a mismatch, nothing is written. The transaction hash goes into `mismatchedFillTxHashes`, the set the tick loop builds per tick, so that the ERC721 `Transfer` emitted by the same transaction cannot relabel this order `invalid` a moment later (see `applyTransfer`). The method still returns `true`, because the log was recognisably ours, which is what `counts.matched` on the health endpoint means. The order is left `active`, as the error message says, "for a human to look at", but there is no flag column or queue, only the error log line.

```ts
// src/components/marketplacev2/poller/application/seaport-event-application.service.ts
    const fromStatus = order.status;
    const buyer = decoded.args.recipient as string;
    const filledAt = typeof log.blockTimestamp === 'number' ? new Date(log.blockTimestamp * 1000) : new Date();

    const gasInfo = await this.fetchGasInfo(log.transactionHash, orderHash);
```

5. The buyer is `recipient`, never inferred from the `Transfer` log. Because ethers version 6 already checksums decoded addresses, `filledBy` is stored checksummed. One Seaport subtlety worth knowing: when a fill goes through Seaport's match functions (`matchOrders` and `matchAdvancedOrders`), Seaport, as far as its source goes, emits `OrderFulfilled` with `recipient` set to the zero address, and an aggregator can pass itself as the recipient. In either case `filledBy` and the transaction row's `buyer` would not be the end user's wallet, and "my purchases" would not show the sale to them.
6. `filledAt` is the block timestamp when the tick loop supplied one, otherwise processing time. See `08` for why the timestamp fetch can fail for a whole tick.
7. `fetchGasInfo` fetches the transaction receipt for `gasUsed` and `gasPrice` (in ethers version 6, the receipt's `gasPrice` is the effective price actually paid). It runs outside the database transaction on purpose, because holding a pooled connection open across an RPC round trip is a classic way to exhaust a pool. It never throws: any failure, or a missing receipt, degrades to `null` gas fields with a warning. It only runs for logs that already matched and verified, so other marketplaces' traffic never costs a receipt fetch.

```ts
// src/components/marketplacev2/poller/application/seaport-event-application.service.ts
    const result = await this.orderRepository.manager.transaction(async (manager) => {
        const updateResult = await manager.update(
            OrderEntity,
            { orderHash, status: In(FILLABLE_FROM_STATUSES) },
            { status: OrderStatus.FILLED, fillTxHash: log.transactionHash, filledAt, filledBy: buyer }
        );
        if (!updateResult.affected) {
            return updateResult;
        }
        await this.transactionService.record(manager, {
            transactionType: TransactionType.SALE,
            orderHash,
            txHash: log.transactionHash,
            blockNumber: String(log.blockNumber),
            occurredAt: filledAt,
            chainId: MARKETPLACEV2_CHAIN_ID,
            tokenContract: order.tokenContract,
            tokenId: order.tokenId,
            domainName: order.domainName,
            tld: order.tld,
            buyer,
            seller: order.maker,
            priceUsdt: order.priceUsdt,
            feeUsdt: order.feeUsdt,
            sellerUsdt: order.sellerUsdt,
            gasUsed: gasInfo.gasUsed,
            effectiveGasPrice: gasInfo.effectiveGasPrice
        });
        const userId = await this.resolveOrderUserId(order, `OrderFulfilled for ${orderHash}`);
        if (userId) {
            await this.listingStatusService.upsertUnlisted(manager, userId, order.domainName, order.tokenId);
            await this.listingHistoryService.record(manager, { userId, domainName: order.domainName, tokenId: order.tokenId, tokenContract: order.tokenContract, orderHash, eventType: ListingEventType.SOLD, txHash: log.transactionHash, occurredAt: filledAt });
        }
        return updateResult;
    });
```

8. Inside one transaction, flip the order to `filled`, but only if its current status is in `FILLABLE_FROM_STATUSES`.

```ts
// src/components/marketplacev2/poller/application/seaport-event-application.service.ts
const FILLABLE_FROM_STATUSES = [OrderStatus.ACTIVE, OrderStatus.CANCELLED, OrderStatus.INVALID, OrderStatus.EXPIRED];
```

The comment above it explains why each non active source is legitimate. `cancelled` is reachable because our UI's cancel is off chain only, so the signature still works on chain. `invalid` is reachable because a domain can be moved away and back, or approval revoked and re granted, making an "invalidated" order fillable again. `expired` is reachable because the expiry sweep uses wall clock time while the poller trails the chain, so a fill that landed just before `endTime` can be processed after the sweep already marked the order expired. In every case the chain is the truth and `filled` wins. If zero rows are affected (another writer got there first, or this is a replay racing a second poller), the callback returns without writing anything else.
9. Insert the `SALE` row into `tbl_marketplacev2_transactions`, the "single source of truth for every completed, on chain financial event" per its entity comment, append only, unique on `(orderHash, transactionType)`. Price, fee and seller amounts are copied from the order row, not the event, which is fine because verification just proved they agree.
10. Resolve the seller's `userId`, upsert their listing status row to `UNLISTED`, and insert a `SOLD` history row with the fill transaction hash and on chain time.
11. After commit, log the transition. A transition from `cancelled`, `invalid` or `expired` logs a warning naming it as legitimate; a normal `active` to `filled` logs at info. As `08` explains, the warning only reaches stdout, not CloudWatch, so the most interesting fills (an off chain cancelled order selling anyway) are the least visible ones.

### What a replay does

If the same `OrderFulfilled` is processed again, step 3 returns early because the order is already `filled`. If two pollers race, both may pass step 3; the first `UPDATE` takes the row lock and commits; the second `UPDATE` waits on that lock, and when it proceeds Postgres re evaluates the `WHERE` against the committed row (`READ COMMITTED` semantics), sees `filled` is not in the list, and affects zero rows, so the second transaction writes nothing else. Even if somehow it did reach the inserts, the unique constraints and `ON CONFLICT DO NOTHING` would make them no ops. The only duplicated side effects are log lines and a second receipt fetch.

### The off chain cancel caveat, from the product side

`OrderService.cancel` (`order/order.service.ts`, around line 750) marks the order `cancelled` and returns `offChainOnly: true` with the message "Its signature can no longer be used to fill this order on this marketplace." That is accurate for this marketplace's UI, but the signed order was publicly readable while it was active, and anyone who kept it can still submit it to Seaport directly. The poller handles this honestly by recording the sale, but a seller who "cancelled" may be surprised to find they sold. The only true cancel is an on chain `cancel` call (which the poller does handle) or a counter increment (which it does not).

## OrderCancelled, an on chain cancel

`OrderCancelled(bytes32 orderHash, address indexed offerer, address indexed zone)` is emitted when the offerer (or the order's zone) calls Seaport's `cancel` with the order components. It is the only cancel that truly kills a signature.

```ts
// src/components/marketplacev2/poller/application/seaport-event-application.service.ts
async applyOrderCancelled(log: DecodableLog): Promise<boolean> {
    const decoded = this.decode(log, 'OrderCancelled');
    if (!decoded) return false;

    const orderHash = decoded.args.orderHash as string;
    const occurredAt = typeof log.blockTimestamp === 'number' ? new Date(log.blockTimestamp * 1000) : new Date();

    const touched = await this.orderRepository.manager.transaction(async (manager) => {
        const updateResult = await manager
            .createQueryBuilder()
            .update(OrderEntity)
            .set({ status: OrderStatus.CANCELLED })
            .where('orderHash = :orderHash AND status = :status', { orderHash, status: OrderStatus.ACTIVE })
            .returning(['maker', 'domainName', 'tokenId', 'tokenContract', 'userId'])
            .execute();
        const rows = (updateResult.raw as Array<{ maker: string; domainName: string; tokenId: string; tokenContract: string; userId: string | null }>) ?? [];
        if (rows.length === 0) {
            return rows;
        }
        const [order] = rows;
        const userId = await this.resolveOrderUserId(order, `OrderCancelled for ${orderHash}`);
        if (userId) {
            await this.listingStatusService.upsertUnlisted(manager, userId, order.domainName, order.tokenId);
            await this.listingHistoryService.record(manager, { userId, domainName: order.domainName, tokenId: order.tokenId, tokenContract: order.tokenContract, orderHash, eventType: ListingEventType.CANCELLED, txHash: log.transactionHash, occurredAt });
        }
        return rows;
    });
    ...
}
```

Step by step: decode; open a transaction; run a single `UPDATE ... SET status = 'cancelled' WHERE orderHash = :orderHash AND status = 'active' RETURNING maker, domainName, tokenId, tokenContract, userId`; if no row came back, the order is not ours or not active, so return `false`; otherwise use the returned columns (not a separate read) to resolve the user, upsert listing status to `UNLISTED`, and insert a `CANCELLED` history row that, unlike the off chain cancel's row, carries a real `txHash`.

`RETURNING` is the right tool here. It hands back exactly the row the update changed, atomically, so there is no window between a read and a write in which another writer could change the row. Unlike `applyOrderFulfilled`, there is no read before the transaction at all, which also means every `OrderCancelled` on Seaport, from any marketplace, costs one short transaction with a primary key `UPDATE` that matches nothing. That is cheap per log, but it is a database round trip for every foreign cancel.

Three details deserve a note.

The `where` string writes `orderHash` bare, while every other raw SQL fragment in marketplace v2 quotes camel case columns (`"createdAt"`, `"tokenContract"`). Postgres folds unquoted identifiers to lower case, so `orderHash` would normally mean a nonexistent `orderhash` column. It works here because TypeORM 0.3's update query builder disables alias prefixing and rewrites bare entity property names in the statement into their quoted column names before sending it. That is a library behaviour rather than something the code states, it is inconsistent with the rest of the module, and the specs cannot catch a regression because both the unit spec's mock and the integration spec's fake query builder ignore the SQL text entirely. Quoting it (`'"orderHash" = :orderHash AND status = :status'`) would remove the dependency. If it ever did break, Postgres would raise `42703`, which `08` explains is classified as transient, so the poller would stall on the first range containing any Seaport cancel.

Only `active` orders are eligible. An order already soft cancelled, expired or invalid is left as it is, which is harmless because all of those are already "not listed", but it means the history never records the on chain cancel for those orders.

`occurredAt` is always processing time in practice, because the tick loop passes the raw `log` (with no `blockTimestamp`) for cancels, unlike fills. During catch up after downtime, the `CANCELLED` row's `occurredAt` can be hours later than the actual cancel, and the `ListingHistoryEntity` comment's promise that `occurredAt` is "never poller processing time" does not hold for this event type.

## CounterIncremented, deliberately not handled

`CounterIncremented(uint256 newCounter, address indexed offerer)` fires when a seller calls Seaport's `incrementCounter`, which instantly invalidates every order they ever signed with an older counter, on every marketplace. Our UI never offers it, so it can only come from another marketplace's "cancel all" button.

```ts
// src/components/marketplacev2/poller/application/seaport-event-application.service.ts
async applyCounterIncremented(log: DecodableLog): Promise<boolean> {
    const decoded = this.decode(log, 'CounterIncremented');
    if (!decoded) return false;

    const offerer = decoded.args.offerer as string;
    const newCounter = (decoded.args.newCounter as bigint).toString();
    this.customLoggerService.warn(`CounterIncremented for offerer ${offerer} (newCounter=${newCounter}) - every order this seller signed is now invalid on-chain. NOT handled by this poller (out of scope, see B04_MASTER_PLAN.md). No order rows touched.`);
    return true;
}
```

It writes nothing and returns `true` for every counter bump on Seaport, including the vast majority from sellers who have never used this marketplace, which inflates `counts.matched` on the health endpoint with events that are not ours. The comment calls this an intentional, documented scope cut, and that is fair for a demo, but the consequence is a real ghost listing: the order stays `active` in browse, and every buyer who tries to purchase it has their transaction revert. Because the log is a `warn`, it never reaches CloudWatch. The fix is not hard, since `OrderEntity.counter` stores the signed counter: `UPDATE ... SET status = 'invalid' WHERE maker = :offerer AND status = 'active' AND "counter"::numeric < :newCounter RETURNING ...`, with a new `invalidReason` such as `counter_incremented`.

## Transfer, the ghost listing watcher

`Transfer(address indexed from, address indexed to, uint256 indexed tokenId)` from the domain NFT contract. All three parameters are indexed, so they live in the topics and `data` is empty. This method's whole purpose is closing the ghost listing gap: a seller lists a domain, then moves it to another wallet outside the marketplace. The signed order can no longer be filled, but without this watcher it would sit in browse forever.

```ts
// src/components/marketplacev2/poller/application/domain-event-application.service.ts
async applyTransfer(log: DecodableLog, mismatchedFillTxHashes?: Set<string>): Promise<boolean> {
    const decoded = this.decode(log, 'Transfer');
    if (!decoded) return false;

    const to = decoded.args.to as string;
    const tokenId = (decoded.args.tokenId as bigint).toString();

    const order = await this.orderRepository.findOne({
        where: { tokenContract: this.chainConfig.domainNftAddress, tokenId, status: OrderStatus.ACTIVE }
    });
    if (!order) {
        this.customLoggerService.debug(`Transfer of tokenId ${tokenId} to ${to} - no active order on file, ignoring`);
        return false;
    }

    if (getAddress(to) === getAddress(order.maker)) {
        this.customLoggerService.debug(`Transfer of tokenId ${tokenId} back to its own maker ${order.maker} - not an ownership change, ignoring`);
        return true;
    }

    if (mismatchedFillTxHashes?.has(log.transactionHash)) {
        this.customLoggerService.warn(`Transfer of tokenId ${tokenId} in tx ${log.transactionHash} accompanied a fill that failed verification - NOT marking order ${order.orderHash} invalid, ...`);
        return true;
    }

    const occurredAt = await this.resolveOccurredAt(log);

    if (occurredAt.getTime() <= order.createdAt.getTime()) {
        this.customLoggerService.debug(`... predates order ${order.orderHash}'s createdAt ... - stale replay, not invalidating`);
        return true;
    }

    const result = await this.orderRepository.manager.transaction(async (manager) => {
        const updateResult = await manager.update(OrderEntity, { orderHash: order.orderHash, status: OrderStatus.ACTIVE, createdAt: LessThan(occurredAt) }, { status: OrderStatus.INVALID, invalidReason: OrderInvalidReason.OWNER_CHANGED });
        if (!updateResult.affected) {
            return updateResult;
        }
        const userId = await this.resolveOrderUserId(order, `Transfer invalidation for order ${order.orderHash}`);
        if (userId) {
            await this.listingStatusService.upsertUnlisted(manager, userId, order.domainName, order.tokenId);
            await this.listingHistoryService.record(manager, { /* eventType: INVALIDATED, txHash: log.transactionHash, occurredAt */ });
        }
        return updateResult;
    });
    ...
}
```

1. Decode `to` and `tokenId` (`tokenId` as a decimal string, since it is far larger than a safe JavaScript number).
2. Look for an `active` order for this exact token on the configured contract. The partial unique index `idx_marketplacev2_orders_active_listing_unique` on `(tokenContract, tokenId) WHERE status = 'active'` guarantees there is at most one. The comparison is plain equality on `tokenContract`, which works because both the config and `OrderService.create` store it checksummed. Most `Transfer` logs on the UD contract (mints, ordinary moves of unlisted domains) end here.
3. A transfer to the maker is not an ownership change. Return `true` (the token is ours) without writing.
4. If this transaction's fill failed verification earlier in the same tick, do not invalidate. This is the one case the next step cannot catch, because a failed verification leaves the order `active`.
5. Resolve when the transfer happened on chain. Raw logs from the tick loop have no timestamp for domain events, so `resolveOccurredAt` fetches the block (lazily, only now that every cheap filter has passed). Crucially, a failed fetch throws instead of falling back to processing time; commit `96f221ec`'s whole point was that the fallback made the next guard impossible to trigger. That throw has no code, so the tick loop treats it as transient and retries the range.
6. The stale replay guard. If the transfer happened at or before the order was created, it cannot be the reason this order is invalid, it is an old event being replayed after catch up or a retry. Example: a seller moved the domain out at block 100 and back at block 120, then listed it; the poller, catching up, reaches block 100 after the listing exists. Without this guard, the old move would invalidate the new listing.
7. Inside a transaction, the conditional update repeats both conditions, `status = 'active'` and `createdAt < occurredAt`, so the guard is enforced atomically even if the row changed since step 2. Then side tables: listing status `UNLISTED` and an `INVALIDATED` history row with the transfer's transaction hash and on chain time.
8. Zero affected rows (the order was cancelled, filled or expired between steps 2 and 7) is logged at debug and is not an error.

### The sale is also a transfer

When a domain sells through Seaport, the same transaction emits `OrderFulfilled` and the NFT's `Transfer` from seller to buyer. If the transfer were processed first, step 2 would find the order still `active`, step 3 would see `to` is not the maker, and the order would be marked `invalid` (`owner_changed`), hiding a real sale. The tick loop's rule, every Seaport log in the range before any domain log, prevents that: by the time the transfer is processed the order is already `filled`, so step 2 finds nothing. The team calls this "Named Trap #2", and the integration spec's acceptance case `#8/#10` proves it end to end with real services and a stateful fake database.

### Clocks and time zones in the stale replay guard

The guard compares a block timestamp (chain time) with `order.createdAt`, which `OrderService.create` sets from the application server's clock, stored in a `timestamp` column with no time zone. Two consequences follow. A transfer mined within a second or two of the listing being created can carry a block timestamp at or before `createdAt` if the server clock runs slightly ahead of the validators', and would be treated as stale, leaving a ghost listing. And because `timestamp without time zone` stores whatever wall clock time node postgres serialises in the writing process's local zone, an order created by a process running in a different time zone from the one comparing it (the documented case of a developer's laptop in India pointed at the shared UAT database) would have `createdAt` shifted by the zone offset, five and a half hours for IST, and every transfer in that window after listing would be wrongly skipped as stale. Using `timestamptz` for `createdAt`, as the shared `BaseEntity` elsewhere in this codebase already does, would remove the second risk entirely.

### One way doors

Invalidation is never reversed. If a seller moves a domain away and back, the order is `invalid` in the database but fillable on chain again. That is mostly fine, because a fill would still be recorded ("chain wins"), but the listing has vanished from browse, so in practice nobody will find it to fill it. The same is true of approval revocation followed by re approval.

## ApprovalForAll, the seller revoked Seaport

`ApprovalForAll(address indexed owner, address indexed operator, bool approved)`. Our orders use no conduit (`OrderService` rejects any `conduitKey` other than the no conduit key, around `order.service.ts:277`, and checks `isApprovedForAll(offerer, seaportAddress)` at create), so Seaport itself must be an approved operator for the seller's whole UD collection. Revoking that approval makes every one of the seller's orders unfillable at once.

```ts
// src/components/marketplacev2/poller/application/domain-event-application.service.ts
async applyApprovalForAll(log: DecodableLog): Promise<boolean> {
    const decoded = this.decode(log, 'ApprovalForAll');
    if (!decoded) return false;

    const owner = decoded.args.owner as string;
    const operator = decoded.args.operator as string;
    const approved = decoded.args.approved as boolean;

    if (getAddress(operator) !== getAddress(this.chainConfig.seaportAddress)) {
        return false;
    }
    if (approved) {
        return false;
    }

    const ownerAddress = getAddress(owner);
    const occurredAt = await this.resolveOccurredAt(log);

    const affected = await this.orderRepository.manager.transaction(async (manager) => {
        const updateResult = await manager
            .createQueryBuilder()
            .update(OrderEntity)
            .set({ status: OrderStatus.INVALID, invalidReason: OrderInvalidReason.APPROVAL_REVOKED })
            .where('maker = :maker AND status = :status AND "createdAt" < :occurredAt', { maker: ownerAddress, status: OrderStatus.ACTIVE, occurredAt })
            .returning(['orderHash', 'tokenId', 'domainName', 'tokenContract', 'userId'])
            .execute();
        const touchedOrders = (updateResult.raw as Array<{ orderHash: string; tokenId: string; domainName: string; tokenContract: string; userId: string | null }>) ?? [];
        for (const order of touchedOrders) {
            const userId = await this.resolveOrderUserId({ maker: ownerAddress, userId: order.userId }, `ApprovalForAll revocation from ${ownerAddress}`);
            if (userId) {
                await this.listingStatusService.upsertUnlisted(manager, userId, order.domainName, order.tokenId);
                await this.listingHistoryService.record(manager, { /* eventType: INVALIDATED for this order */ });
            }
        }
        return touchedOrders;
    });
    ...
}
```

1. Decode. `approved` is not indexed, so it is decoded from `data`, never filtered on in the `getLogs` topics, exactly as the ABI comment warns.
2. Ignore approvals for any operator other than Seaport (another marketplace's conduit, for example).
3. Ignore grants. Every seller's first listing begins with `setApprovalForAll(seaport, true)`, so grants are common and harmless.
4. Checksum the owner, because `maker` is stored checksummed and the `WHERE` uses plain equality.
5. Fetch the block timestamp, throwing on failure, as with transfers.
6. One bulk `UPDATE` of every `active` order by this maker created before the revocation, with `RETURNING` the rows it actually changed.
7. For each returned row, resolve the user, upsert listing status and insert an `INVALIDATED` history row.

The comment above the update explains a bug the team fixed in commit `b8c0f074`: an earlier version ran `find()` first and then looped over that snapshot to write history rows, so an order cancelled or filled between the read and the update would get a wrong `INVALIDATED` history row for a write that never touched it. Basing the loop on `RETURNING` makes the history exactly match what changed. The `"createdAt" < :occurredAt` condition is the same stale replay guard as transfers, added in `96f221ec`, with the same clock and time zone caveats.

There is no `tokenContract` condition in the `WHERE`. That is correct today because every order in the table is for the one configured UD contract and only that contract's logs are fetched, but if marketplace v2 ever lists a second collection, a revocation on one collection would invalidate the seller's listings on the other.

## Idempotency, event by event

The poller delivers each log at least once. Replays happen after a crash between applying logs and advancing the cursor, after any transient abort (the whole range is retried), after a cursor regression caused by two pollers, and whenever two pollers overlap. Here is why each event survives a replay.

| Event | First guard | Atomic guard | Side table guard | Replay result |
|---|---|---|---|---|
| `OrderFulfilled` | Early return if already `filled` | `WHERE status IN (active, cancelled, invalid, expired)` | `SALE` unique on `(orderHash, transactionType)`; `SOLD` unique on `(orderHash, eventType)`; both `ON CONFLICT DO NOTHING` | No writes, warning log |
| `OrderFulfilled`, verification mismatch | none | nothing written | nothing written | Same error log again |
| `OrderCancelled` | none | `WHERE status = 'active'` with `RETURNING` | `CANCELLED` unique | Zero rows, debug log |
| `CounterIncremented` | none | nothing written | nothing written | Same warning again |
| `Transfer` | `findOne` only `active`; stale replay check | `WHERE status = 'active' AND "createdAt" < occurredAt` | `INVALIDATED` unique | Zero rows or not found |
| `ApprovalForAll` | operator and `approved` filters | `WHERE maker AND status = 'active' AND "createdAt" < occurredAt` with `RETURNING` | `INVALIDATED` unique | Zero rows |

Two subtleties. First, the history uniqueness on `(orderHash, eventType)` means an order can only ever have one row per event type. That is fine for the poller's events, but it also means an order that was invalidated, became fillable again, and was invalidated a second time would only ever show the first `INVALIDATED`. Second, idempotency here protects database state, not external side effects. The poller sends no emails and calls no webhooks today, unlike the v1 transaction cron in `03/04` which emails buyer and seller from inside its loop. If anyone ever adds a notification to these methods, it must go after the `affected` check, or it will fire on every replay.

## Ordering guarantees, and what they do and do not promise

Within each `getLogs` result, nodes return logs ordered by block number and then log index. That ordering is universal in practice, though not formally part of the JSON RPC spec. The tick loop then processes the whole Seaport list, then the whole domain list, so the effective order is "all Seaport events in the range in chain order, then all domain events in the range in chain order". That is not chronological across the two contracts. Here is why it is still safe for the state machine, and where it is merely cosmetic.

It is safe for sales because `filled` is reachable from every other status and is terminal. Whatever a domain event did or did not do, a fill in the same range wins, so processing fills first gives the same final status as processing in true order, and additionally prevents the sale's own transfer from misfiring.

It is safe for transfers and revocations across different blocks because both are guarded by `createdAt < occurredAt` using the block's own timestamp, not by the order of processing, and because both only act on `active` orders.

It differs from chronological order cosmetically when an on chain cancel and an invalidating domain event affect the same order in one range. If a seller transfers the domain away at block 100 and then cancels the now unfillable order on chain at block 110, chronological processing would mark it `invalid` and then ignore the cancel; Seaport first processing marks it `cancelled` (with a `CANCELLED` history row) and then the transfer finds no active order. Both outcomes are "not listed", but the reason shown in history differs depending on whether the two events happened to fall in the same poll range.

The `mismatchedFillTxHashes` set is rebuilt each tick, which is sufficient, because a fill and its paired transfer are in the same transaction, therefore the same block, therefore always in the same range, even when the range is shrunk.

## Living alongside the other writers

The poller is not the only process writing `tbl_marketplacev2_orders`, and the conditional updates are what keep the writers from trampling each other.

`OrderService.cancel` runs `UPDATE ... WHERE orderHash AND status = 'active'`, so it cannot cancel a filled order, and a later fill of a soft cancelled order still succeeds because `cancelled` is fillable from. `OrderService.expireIfStale`, run by `ListingExpiryScheduler`, runs `UPDATE ... WHERE "endTime" <= now AND status = 'active'`, and a later fill still wins from `expired`. The scheduler shares the `POLLER_ENABLED` flag so that an environment never sweeps without also polling, because sweeping ahead of a lagging poller is exactly what produces the `expired` to `filled` transition. A fill and an expiry racing on the same row are serialised by the row lock, and whichever commits second re evaluates its `WHERE`.

Two pollers racing each other are serialised the same way, which is why the database state survives the multiple process scenarios described in `08`, even though the cursor does not.

## Bugs and risks found in this layer

| Where | Risk | Concrete failure |
|---|---|---|
| `application/seaport-event-application.service.ts:301-309` | `CounterIncremented` is logged, never applied | A seller bumps their counter on another marketplace; their listings stay `active` here and every buyer's transaction reverts |
| `application/seaport-event-application.service.ts:307` | That alert is a `warn`, which never reaches CloudWatch | Nobody sees it in production |
| `application/seaport-event-application.service.ts:308` | Returns `true` for every counter bump on Seaport | `counts.matched` on the health endpoint is inflated by other marketplaces' sellers |
| `application/seaport-event-application.service.ts:103-108` | Mismatched fills only produce a log line, no flag or queue | A sold domain can stay `active` indefinitely, and nobody is prompted to look |
| `application/seaport-event-application.service.ts:111` | `filledBy` is Seaport's `recipient` | Match style fills (zero address recipient) or aggregators acting as recipient record the wrong buyer, so the sale is missing from the real buyer's purchases |
| `application/seaport-event-application.service.ts:350` | Fee recipient verified against the current config | Rotating `FEE_RECIPIENT` makes every older active order fail verification at sale time |
| `application/seaport-event-application.service.ts:214` | Bare `orderHash` in raw SQL relies on TypeORM rewriting | A TypeORM behaviour change turns every Seaport cancel into a `42703` error, which the tick classifies as transient, stalling the poller |
| `application/seaport-event-application.service.ts:207` with `tick/chain-event-source-poller.service.ts:358` | Cancels never receive a block timestamp | `CANCELLED` history `occurredAt` is processing time, hours off during catch up |
| `application/seaport-event-application.service.ts:209-216` | A transaction per foreign cancel | Every other marketplace's on chain cancel costs a database round trip |
| `application/domain-event-application.service.ts:147, 153, 226` | Stale replay guard compares chain time with server time in a `timestamp` (no zone) column | Clock skew of a second or two, or an order created by a process in another time zone, makes a genuine transfer look stale, leaving a ghost listing |
| `application/domain-event-application.service.ts:226` | No `tokenContract` condition on revocation | Fine today; wrong the day a second collection is listed |
| `application/domain-event-application.service.ts:66-72` | Block fetched per log, no cache, and a missing block throws a code less error | Several domain events in one block repeat the same RPC call; a node that cannot find the block wedges the poller on that range (transient classification) |
| `application/domain-event-application.service.ts:153`, `:225` | Invalidation is one way | A domain moved out and back, or approval revoked and re granted, stays hidden from browse even though it is fillable |
| `application/*.service.ts` `resolveOrderUserId` | Legacy fallback runs outside the transaction's manager and is copy pasted three times | Two pool connections per legacy event; the three copies can drift |
| `listing-status/entity/listing-history.entity.ts:14` | One history row per `(orderHash, eventType)` | A second invalidation of the same order is silently not recorded |
| `order/order.service.ts` `cancel` (around line 750) | Off chain cancel does not kill the signature | A seller who "cancelled" can still be sold out from under them; the poller records it as a fill |

## The tests, file by file

Every test in this folder is a plain Jest unit test that builds the class with `new` and hand made mocks; none uses `@nestjs/testing`, none touches a real database or RPC. The provider fields are created inside constructors with no injection seam, so every spec overrides the private `provider` (or `fetchBlockTimestamp`) after construction, the same technique the order specs use.

### `abi/assert-event-topics.spec.ts` (8 tests)

`TC2.1` proves the Seaport fragments are byte for byte the spike's, by comparing against an inline copy. `TC2.2` boots the real ABI and checks the three pinned topic hashes plus that the two unpinned ones are well formed 32 byte hex, and that exactly one debug message mentioning both was emitted. `TC2.3` mutates `SpentItem` into the five field `OfferItem` shape and expects a `PollerTopicAssertionError` naming `OrderFulfilled`, which is Trap 1 in test form. `TC2.4` and `TC2.5` change a parameter type in `Transfer` (`uint256` to `uint128`) and `ApprovalForAll` (`bool` to `uint8`). The `TC2.4` comment records a nice lesson: an earlier version only moved `indexed` and never threw, because `indexed` is not part of the canonical signature and so does not change the topic hash. `TC2.6` proves a mutated `OrderCancelled` does not throw, pinning the documented gap. `TC2.7` restates that all of this runs without booting anything. The last test removes `OrderFulfilled` entirely and expects the "is not present" error.

### `poller.service.spec.ts` (4 tests)

`TC1.2` and its sibling check `health()` returns a `null` cursor when no row exists and the summary when one does. The seeding tests assert the SQL contains `ON CONFLICT ("id") DO NOTHING`, that the parameters are `[1, '59999999']` for a start block of 60,000,000 (start minus one), and that calling `onModuleInit` twice issues the same idempotent insert twice. Nothing tests that a topic assertion failure actually fails `onModuleInit`.

### `tick/chain-event-source-poller.service.spec.ts` (30 tests)

The builder mocks the cursor repository's `findOne` and `update`, both application services (pushing `'seaport'` or `'domain'` into a shared `callOrder` array), and the provider's `getBlockNumber`, `getLogs` and `getBlock`. Log fixtures only set `topics[0]`, since the application services are mocked.

Range math: `TC6.1` (inside the confirmations buffer, no `getLogs`, no cursor write), `TC6.3` (far behind, the range is exactly `MAX_RANGE` blocks), and a close to head case (capped at `head - CONFIRMATIONS`).

Processing order and cursor: `TC6.2` asserts `callOrder` is `['seaport', 'domain']` and the cursor is written once with `head - CONFIRMATIONS`; two dispatch tests check each topic reaches its handler.

RPC failures: `TC6.6` (a non retryable `getLogs` error warns naming the range and leaves the cursor), a chain head failure (error log, no cursor write), `TC6.4` (an `ETIMEDOUT` that recovers still advances; this test really waits through a backoff, hence its fifteen second timeout), and `TC6.5` (a persistent 503 exhausts retries, about five seconds of real sleeping).

Application failures, the newest group: `TC6.7` (a Postgres `22001` "value too long" on one cancel is skipped and the cursor still advances), a `BAD_DATA` fill whose transaction hash must appear in the set passed to `applyTransfer`, a code less "Connection terminated" error that stops the tick before any domain log and logs the transient marker, the same for a domain log with `ECONNRESET` (note that this is a plain `Error` whose message is `ECONNRESET`, not a `code`, so it is transient only because it has no code at all), a transient failure that clears on the next tick and replays the range, and a check that the per tick range line is `debug`, not `log`.

Concurrency: `TC6.8` starts two ticks while the first is blocked on an unresolved `getBlockNumber` promise and proves the second never calls the provider; a follow up proves sequential ticks both run.

`getLag`: `TC6.9` and `TC6.10`. Lifecycle: `TC6.11` (double `start()` creates one interval), `TC6.12` (no tick after `stop()`, with fake timers), and the two `POLLER_ENABLED` tests. `getStatus`: `healthy` false before any tick, true after a fresh tick within threshold, false with lag over 300, false with a stale `lastTickAt`, counts from the last tick only (`matched: 3` because the mocks always return `true`), and counts reset on an empty tick.

Not covered: the range shrinking loop and `isRangeTooLargeError` (no test at all), `fetchFillBlockTimestamps` failing, `readCursor` throwing, the `.catch` in `start()`, and anything about two processes.

### `application/seaport-event-application.service.spec.ts` (27 tests)

Fixtures are real ABI encoded logs built with `Interface.encodeEventLog`, so decoding is exercised for real; only the repository, manager, query builder, wallet repo, the three side services and the receipt provider are mocked.

For `applyOrderFulfilled`: `TC4.1` (the three argument `manager.update(OrderEntity, criteria, partial)` with `filled`, `fillTxHash`, `filledBy`, a `Date` `filledAt`); the `SALE` record with gas from the receipt (`'21000'` and `'30000000000'`); null gas on a receipt failure and on a missing receipt, with the fill still written; `TC4.2` and `TC4.10` (already filled: no update, warning only); `TC4.3` (unknown hash, debug only); `TC4.4` to `TC4.7` (seller amount, fee recipient, token identifier and short consideration mismatches, each proving no update, and `TC4.4` also proving no receipt fetch and no transaction write); `TC4.7b` (a third consideration item, the tip case, still fills); `TC4.8`, `TC4.9` and the expired case (warnings naming `cancelled -> filled`, `invalid -> filled`, `expired -> filled`, and that the `In` operator's value contains `expired`); `TC4.15` (buyer is exactly `recipient`); the `SOLD` history and `UNLISTED` status writes; and the no user found case (fill still written, side tables skipped, warning).

For `applyOrderCancelled`: `TC4.11` asserts the exact `where` string and parameters, which is the only place the bare `orderHash` SQL appears in a test, as a string comparison rather than against a database; `TC4.12` and `TC4.13` (zero returned rows, no error, debug); and the `CANCELLED` history row with a real transaction hash. `TC4.14` checks the counter warning names the offerer and `newCounter=5`. Four "matched reporting" tests pin the return values.

A small oddity: the helper at the bottom named `getChecksummed` actually lowercases, so `TC4.15` is a case insensitive comparison rather than a checksum check. The behaviour tested is right; the name is misleading.

### `application/domain-event-application.service.spec.ts` (23 tests)

`applyTransfer`: `TC5.1` (the update criteria include `createdAt` with a `lessThan` operator), the history and status writes, `TC5.2`, `TC5.3` and `TC5.10` (no active order, no update), `TC5.4` (transfer to the maker), a stale replay (block an hour before `createdAt`, no update, returns `true`), a fresh transfer (an hour after, updates), a log carrying `blockTimestamp` skips the block fetch, no block fetch when there is no order or the transfer is to the maker, a failed block fetch rejects rather than falling back, and a zero affected race with no error and no info log.

`applyApprovalForAll`: `TC5.5` (the exact `where` string including `"createdAt" < :occurredAt`), the `RETURNING` based loop writing two history rows for two returned orders, the fetched block timestamp used as `occurredAt`, `TC5.6` (a grant does nothing), `TC5.7` (a different operator does nothing), `TC5.8` and `TC5.9` (zero rows, no error). Four matched reporting tests pin the return values.

Not covered here: the `mismatchedFillTxHashes` early return in `applyTransfer` is only covered indirectly by the tick spec (which checks the set is passed) and not by a direct test of the skip.

### `poller.integration.spec.ts` (6 tests)

This is the most valuable file, and its header is unusually honest about what it does not prove. It wires the real `ChainEventSourcePoller`, `SeaportEventApplicationService` and `DomainEventApplicationService` together, replacing only Postgres with a small stateful `FakeRepository` (so a write in one tick is visible in the next) and the RPC with controllable fakes, while the listing status, history and transaction services are plain mocks.

The six cases map onto the brief's acceptance criteria: `#1/#3` a sale fills with the right buyer and hash, and reprocessing the same range leaves the row deep equal to before; a sale writes exactly one `UNLISTED` and one `SOLD` call, still one after a replay; `#7` a soft cancelled order that fills on chain ends `filled` with the warning; `#8/#10` a lone transfer invalidates (`owner_changed`), while a fill and its transfer in the same tick end `filled` with a null `invalidReason`, which is the real end to end proof of the ordering rule; `#9/#11` a revocation invalidates both of a seller's listings and a later grant changes nothing; and `#4/#5` a broken `getLogs` leaves the order active and the next tick retries the same range and fills it.

The fakes have limits worth knowing before you lean on this file. `FakeUpdateQueryBuilder.where` ignores the SQL string and uses the parameter keys as criteria, translating `occurredAt` into `createdAt: LessThan(...)`, so it cannot catch a wrong or unquoted column. `manager.transaction` just calls the callback, with no rollback, so a throw halfway through a callback would leave earlier fake writes applied, the opposite of Postgres, and atomicity is not proven. There is no row locking, so concurrent writers are not simulated. And because the side services are mocks, the `ON CONFLICT DO NOTHING` behaviour of the real inserts is asserted only by call counts, which pass because the order update's `affected` check stops the second call, not because the database ignored a duplicate.

### Supporting specs outside the folder

`src/@core/utils/retry/retry-with-backoff.util.spec.ts` (5 tests) covers success first time, success after two 429s, rethrow after exhausting `maxRetries` (three calls for `maxRetries: 2`), immediate rethrow of a non retryable error, and a custom predicate. There is no spec for `isRetryableChainError` itself. `src/components/marketplacev2/health-routes.guards.spec.ts` pins the poller controller's guard lists by reflection, as described in `08`.

## Frontend note

The statuses the poller writes map directly onto UI states, and the reasons are worth showing rather than flattening. An `invalid` order with `invalidReason: owner_changed` means the seller moved the domain; with `approval_revoked`, the seller withdrew Seaport's permission and every one of their listings went at once. A `filled` order that previously showed `cancelled` is a real sale of a listing the seller thought they had cancelled, and deserves a clear explanation on the seller's history screen. And a listing that stays `active` while purchases keep failing on chain is the `CounterIncremented` gap: if your UI sees a buyer's Seaport transaction revert with an invalid order error, telling the user "this listing is no longer valid on chain" is more honest than a generic failure, because the backend genuinely does not know yet.
