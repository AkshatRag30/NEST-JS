# 05. Cancel, Expiry, and the Listing Expiry Scheduler

## How an active order stops being active

File `04` covered how a row enters `tbl_marketplacev2_orders` as `active`. This file covers the ways the backend itself moves it out again. There are two. The seller can cancel it (`POST /:orderHash/cancel`, a "soft," off chain cancel), or time can run out, in which case `OrderService.expireIfStale` flips it to `expired`, either inline when someone tries to relist that token or periodically from `ListingExpiryScheduler`. The other two exits, `filled` and `invalid`, belong to the poller and are covered in that part of the cluster.

Running underneath all of it is one rule that every read in the marketplace agrees on, the "servable" rule, which decides whether an order should be shown to buyers and whether its signature may be handed out. This file explains that rule first, because cancel and expiry only make sense in its light.

## The servable rule

```ts
// src/components/marketplacev2/order/util/servable-order.util.ts
export function applyServableWhere(qb: SelectQueryBuilder<OrderEntity>, alias: string): void {
    const nowSeconds = Math.floor(Date.now() / 1000).toString();
    qb.andWhere(`${alias}.status = :servableStatus`, { servableStatus: OrderStatus.ACTIVE }).andWhere(`${alias}.endTime > :nowSeconds`, { nowSeconds });
}

export function isOrderServable(order: Pick<OrderEntity, 'status' | 'endTime'>): boolean {
    if (order.status !== OrderStatus.ACTIVE) {
        return false;
    }
    try {
        return BigInt(order.endTime) > BigInt(Math.floor(Date.now() / 1000));
    } catch {
        return false;
    }
}
```

An order is servable when its status is `active` and its `endTime` is strictly in the future. That is exactly the condition under which Seaport itself would accept a fill (Seaport rejects when the block timestamp is at or after `endTime`), so "servable" really means "would probably succeed on chain right now, as far as the database knows."

The header comment makes the key point: "status alone is not enough." The expiry sweep does eventually flip dead rows to `expired`, but "that is a sweep running on its own schedule, not a guarantee that holds at every instant," so the `endTime` comparison "stays the real source of truth." In other words, the status column is a lagging cache of reality, and every read that matters double checks the clock.

There are two versions of the same rule, deliberately kept in one file "so the two forms of the same rule can never drift apart." `applyServableWhere` adds SQL `WHERE` clauses to a TypeORM query builder, for reads that filter many rows in Postgres: browse, stats, the listable domains anti join, and the live order lookup in `myDomains` (`order.service.ts:852`). `isOrderServable` evaluates the same rule in JavaScript against one row already in hand, used by the public order detail route and by the watchlist service. Both compute "now" with `Math.floor(Date.now() / 1000)` in the Node process, and the JavaScript version wraps the `BigInt` conversion in a `try` so a corrupt `endTime` reads as "not servable" rather than throwing. `endTime` is a `bigint` column and the parameter is a string, which Postgres compares numerically after an implicit cast.

### Where the rule protects the signature

```ts
// src/components/marketplacev2/order/order.controller.ts:398
@Get(`:orderHash(${ORDER_HASH_ROUTE_PATTERN})`)
async getOrder(@Param('orderHash') orderHash: string): Promise<Response> {
    const order = await this.orderService.findByHash(orderHash);
    return new Response('Order found', this.toPublicOrder(order));
}

private toPublicOrder(order: OrderEntity): OrderEntity {
    if (this.isServable(order)) {
        return order;
    }
    // Signature intentionally blanked, not omitted - keeps the response
    // shape stable for callers while making it unusable for a fill.
    return { ...order, signature: '' };
}
```

`GET /api/v1/marketplacev2/orders/:orderHash` is unauthenticated "by design (anyone with an orderHash can look up its public details)," and it returns the entire entity, including `rawOrder` and, for a servable order, the `signature`. That is intentional: a buyer needs both to call Seaport's `fulfillOrder`. For any order that is not servable (cancelled, filled, expired, invalid, or `active` but past its `endTime`), the signature is replaced with an empty string. The comment on the route explains why: "a cancelled order's signature must not stay fetchable and fillable forever." Blanking rather than deleting the key keeps the response type stable for TypeScript callers. An unknown but well formed hash gives `404` `"No order found for hash 0x..."` from `findByHash` (`order.service.ts:731`); a malformed one never reaches the controller, because the route regex fails and Express answers `404` itself.

Hold on to this route, because it is the crux of the biggest limitation of soft cancel below.

## Cancelling an order

### The route

```ts
// src/components/marketplacev2/order/order.controller.ts:377
@ApiBearerAuth('defaultBearerAuth')
@Post(`:orderHash(${ORDER_HASH_ROUTE_PATTERN})/cancel`)
@UseGuards(AccessTokenGuard, RequireVerifiedWalletGuard)
async cancelOrder(@Param('orderHash') orderHash: string, @Req() req: Request): Promise<Response> {
    return new Response('Order cancelled', await this.orderService.cancel(orderHash, req.walletAddress, req.user['userId']));
}
```

`POST /api/v1/marketplacev2/orders/:orderHash/cancel` takes no body. The order hash comes from the URL and is constrained by `ORDER_HASH_ROUTE_PATTERN` (`0x` plus 64 hex characters). The same two guards as create apply: a valid access token, and a wallet verification at most one hour old, which produces `req.walletAddress`. The wallet requirement is what makes the maker check below meaningful; without it, anyone logged in could claim to be any wallet.

### The service, line by line

```ts
// src/components/marketplacev2/order/order.service.ts:750
async cancel(orderHash: string, walletAddress: string, userId: string): Promise<CancelOrderResult> {
    const order = await this.findByHash(orderHash);

    if (getAddress(order.maker) !== getAddress(walletAddress)) {
        throw new ForbiddenException(`Wallet ${walletAddress} is not the maker of order ${orderHash}.`);
    }
    if (order.status === OrderStatus.FILLED) {
        throw new ConflictException(`Order ${orderHash} has already been filled and cannot be cancelled.`);
    }

    await this.orderRepository.manager.transaction(async (manager) => {
        const result = await manager.update(OrderEntity, { orderHash, status: OrderStatus.ACTIVE }, { status: OrderStatus.CANCELLED });
        if (!result.affected) {
            return;
        }
        await this.listingStatusService.upsertUnlisted(manager, userId, order.domainName, order.tokenId);
        await this.listingHistoryService.record(manager, {
            userId,
            domainName: order.domainName,
            tokenId: order.tokenId,
            tokenContract: order.tokenContract,
            orderHash,
            eventType: ListingEventType.CANCELLED,
            txHash: null,
            occurredAt: new Date()
        });
    });

    return {
        orderHash,
        status: OrderStatus.CANCELLED,
        offChainOnly: true,
        message: 'This listing has been cancelled off-chain. Its signature can no longer be used to fill this order on this marketplace.'
    };
}
```

1. Load the order or throw `404` `NotFoundException` `"No order found for hash ..."`.
2. Ownership: both sides go through `getAddress` so letter case cannot matter. A different wallet gets `403` `ForbiddenException` with the standard Nest body `{ "statusCode": 403, "message": "Wallet 0x... is not the maker of order 0x....", "error": "Forbidden" }`. Note this is a plain string exception, not the `{ code, message }` shape create uses.
3. A `filled` order cannot be cancelled: `409` `ConflictException`, and no write is attempted. You cannot take back a sale.
4. Inside one transaction, a conditional update: set `cancelled` only where `orderHash` matches and `status` is still `active`. This is the same "the `WHERE` clause is the lock" pattern v1's buy flow uses. If the poller filled the order between step 1 and step 4, the `WHERE` matches nothing and nothing is overwritten.
5. If zero rows were affected, return early without touching listing status or history. The doc comment explains this is deliberate: an order already `cancelled` or `invalid` takes exactly the same zero affected path as a genuine race against a concurrent request, and both are treated as a harmless idempotent success rather than an error. A double click on "cancel" is therefore safe.
6. If one row was affected, flip the user's `tbl_marketplacev2_listing_status` row to `UNLISTED` and append a `CANCELLED` row to `tbl_marketplacev2_listing_history` with `txHash: null` (null because nothing happened on chain), all in the same transaction as the status change.
7. Return the fixed `CancelOrderResult` shape.

The success response is `{ "message": "Order cancelled", "result": { "orderHash": "0x...", "status": "cancelled", "offChainOnly": true, "message": "This listing has been cancelled off-chain. ..." } }`. The interface comment insists on always this exact shape, with no wording ever implying an on chain action happened, and the spec even asserts the message does not match `/cancelled on chain/i`.

### What "soft" cancel really means

Here is the honest truth about this design, and the most important thing in this file for anyone building the seller UI. A Seaport order is a signed promise. The only parties who can revoke it on chain are Seaport itself, when the seller calls `cancel([orderComponents])` or `incrementCounter()` in a transaction. The backend flipping a database row does neither. After a soft cancel, the order is still perfectly valid on Seaport until its `endTime`.

What soft cancel does achieve is that the backend stops advertising the order (it falls out of browse, stats and watchlists because it is no longer servable) and stops handing out its signature (`GET /:orderHash` now blanks it). For a buyer who never saw the signature, the order is effectively gone.

What it cannot achieve is recalling a signature that already left the server. Every buyer who opened the listing's detail while it was active received the signature in that response, and so did any bot scraping the public endpoint. Concretely: Alice lists `alice.crypto` for 100 USDT. Bob opens the listing page, so his browser now holds the signed order. Alice realises the price is too low, cancels, and relists at 500. Ten minutes later Bob clicks "buy" on the page he still has open. His frontend sends the old order and signature straight to Seaport, Seaport checks the signature, the counter, the approval and the time window, all fine, and Bob gets the domain for 100. Seaport emits `OrderFulfilled`, and the poller, whose `FILLABLE_FROM_STATUSES` (`poller/application/seaport-event-application.service.ts:31`) explicitly includes `cancelled`, moves the row to `filled`. Alice's new 500 USDT listing is then for a token she no longer owns, and the poller's domain watcher marks it `invalid` with `owner_changed`.

The response message, "Its signature can no longer be used to fill this order on this marketplace," is technically defensible (this marketplace will not serve it) but easy for a seller to misread as "nobody can buy it now." A safer seller experience would offer a "cancel on chain" button that calls Seaport's `cancel` with the stored `rawOrder` (one cheap transaction), or at least warn sellers relisting at a higher price that the old order remains fillable until it expires. The frontend's buy flow reads Seaport's `getOrderStatus` before filling, but a soft cancel never sets Seaport's `isCancelled` flag, so that check does not help here.

## Expiring stale orders

### Why expiry needed to be written at all

You might expect an order whose `endTime` has passed to need no action, since Seaport will refuse it anyway and the servable rule hides it from buyers. The doc comment on `expireIfStale` explains the bug that proved otherwise. The partial unique index `idx_marketplacev2_orders_active_listing_unique` only allows one `active` row per token, and, as the comment puts it, that index has no escape hatch like the servable rule's clock check. A row whose window had closed was still `active` as far as the index was concerned, so "every relist of an expired domain fail[ed] with a misleading 'already listed' until the seller manually cancelled the dead row." Separately, the seller's own pages (`my-listings` and `my-domains`) read the listing status table, which still said `LISTED`.

### expireIfStale

```ts
// src/components/marketplacev2/order/order.service.ts:551
async expireIfStale(tokenContract: string, tokenId: string): Promise<boolean> {
    const nowSeconds = Math.floor(Date.now() / 1000).toString();
    return this.orderRepository.manager.transaction(async (manager) => {
        const updateResult = await manager
            .createQueryBuilder()
            .update(OrderEntity)
            .set({ status: OrderStatus.EXPIRED })
            .where('"tokenContract" = :tokenContract AND "tokenId" = :tokenId AND status = :status AND "endTime" <= :nowSeconds', {
                tokenContract,
                tokenId,
                status: OrderStatus.ACTIVE,
                nowSeconds
            })
            .returning(['orderHash', 'maker', 'domainName', 'tokenContract', 'tokenId', 'userId'])
            .execute();
        const rows = (updateResult.raw as Array<{ /* ... */ }>) ?? [];
        if (rows.length === 0) {
            return false;
        }
        const [row] = rows;
        const userId = await this.resolveOrderUserId(row, `expireIfStale for order ${row.orderHash}`);
        if (userId) {
            await this.listingStatusService.upsertUnlisted(manager, userId, row.domainName, row.tokenId);
            await this.listingHistoryService.record(manager, {
                userId, domainName: row.domainName, tokenId: row.tokenId, tokenContract: row.tokenContract,
                orderHash: row.orderHash, eventType: ListingEventType.EXPIRED, txHash: null, occurredAt: new Date()
            });
        }
        this.customLoggerService.log(`order ${row.orderHash} active -> expired (endTime passed) - tokenId ${row.tokenId} on ${row.tokenContract}`);
        return true;
    });
}
```

Given one token, this finds the `active` row whose `endTime` is at or before now and flips it to `expired` in a single conditional `UPDATE ... RETURNING` statement. `RETURNING` is a Postgres feature that hands back the updated rows from the same statement, so there is no gap between "find" and "change" for another writer to slip into, and the code gets the order's details without a second query. Note the boundary: expiry uses `endTime <= now` while servable uses `endTime > now`, so the two are exact complements and an order is never both servable and expirable.

If a row was expired, the method resolves which user owns it. `resolveOrderUserId` (`order.service.ts:523`) uses the stored `userId` when present, and falls back to `walletRepo.findByWalletWithoutNetwork(order.maker)` only for legacy rows created before that column existed, logging a warning and returning `null` if no user is found. With a user, it flips their listing status to `UNLISTED` and appends an `EXPIRED` history row (inserted with `orIgnore()` against the `(orderHash, eventType)` unique key, so running twice is harmless). All of it shares one transaction. It returns `true` if it expired something, `false` otherwise.

It is called from two places: inline from `create()` just before check 11 (`order.service.ts:645`), so a seller relisting their own expired domain is never blocked by their own ghost row, and from the batch sweep below.

### expireAllStaleBatch

```ts
// src/components/marketplacev2/order/order.service.ts:593
private static readonly STALE_LISTING_SWEEP_BATCH_SIZE = 200;

async expireAllStaleBatch(failedOrderHashes: Set<string> = new Set()): Promise<number> {
    const nowSeconds = Math.floor(Date.now() / 1000).toString();
    const staleRows = await this.orderRepository.find({
        where: {
            status: OrderStatus.ACTIVE,
            endTime: LessThanOrEqual(nowSeconds),
            ...(failedOrderHashes.size > 0 ? { orderHash: Not(In([...failedOrderHashes])) } : {})
        },
        order: { endTime: 'ASC' },
        take: OrderService.STALE_LISTING_SWEEP_BATCH_SIZE
    });
    let expiredCount = 0;
    for (const row of staleRows) {
        try {
            if (await this.expireIfStale(row.tokenContract, row.tokenId)) {
                expiredCount++;
            }
        } catch (err) {
            failedOrderHashes.add(row.orderHash);
            this.customLoggerService.error(`listing expiry sweep: failed to expire order ${row.orderHash} ... Skipping it for the rest of this sweep.`);
        }
    }
    return expiredCount;
}
```

One batch finds up to 200 stale `active` rows, oldest `endTime` first, and expires each through `expireIfStale`, one small transaction per row. The cap exists so "one scheduler tick can never hold a single giant transaction batch open." Each row is isolated with its own `try`: one row that throws (a deadlock, say) is logged, added to `failedOrderHashes`, and skipped, so it can neither abort the batch nor "hog the front of the queue every pass," because the next batch's query excludes it with `Not(In([...]))`. The set is owned by the caller and shared across every batch of one sweep, then thrown away, so a failed row is retried fresh on the next sweep. The method is on `OrderServiceInterface` (`interface/order-service.interface.ts:162`) so the scheduler can call it through the injection token.

## The listing expiry scheduler

```ts
// src/components/marketplacev2/order/listing-expiry.scheduler.ts
const SWEEP_INTERVAL_MS = 5 * 60 * 1000;

@Injectable()
export class ListingExpiryScheduler implements OnModuleInit, OnModuleDestroy {
    private intervalHandle: ReturnType<typeof setInterval> | null = null;
    private sweepInFlight = false;

    constructor(
        private readonly customLoggerService: CustomLoggerService,
        @Inject('OrderServiceInterface')
        private readonly orderService: OrderServiceInterface,
        @Inject(CHAIN_CONFIG)
        private readonly chainConfig: ChainConfig
    ) {}

    async onModuleInit(): Promise<void> {
        if (!this.chainConfig.pollerEnabled) {
            this.customLoggerService.log('listing expiry sweep disabled - POLLER_ENABLED is false for this environment. Not starting.');
            return;
        }
        await this.start();
    }

    async onModuleDestroy(): Promise<void> {
        await this.stop();
    }

    async start(): Promise<void> {
        if (this.intervalHandle) return;
        this.intervalHandle = setInterval(() => {
            this.sweep().catch((err) => {
                this.customLoggerService.error(`listing expiry sweep: unhandled failure - ${err instanceof Error ? err.message : String(err)}`);
            });
        }, SWEEP_INTERVAL_MS);
        this.customLoggerService.log(`listing expiry sweep started - running every ${SWEEP_INTERVAL_MS}ms`);
    }

    async sweep(): Promise<void> {
        if (this.sweepInFlight) {
            this.customLoggerService.debug('listing expiry sweep skipped - previous sweep still in flight');
            return;
        }
        this.sweepInFlight = true;
        try {
            let totalExpired = 0;
            const failedOrderHashes = new Set<string>();
            let batchExpired: number;
            let failedBefore: number;
            do {
                failedBefore = failedOrderHashes.size;
                batchExpired = await this.orderService.expireAllStaleBatch(failedOrderHashes);
                totalExpired += batchExpired;
            } while (batchExpired > 0 || failedOrderHashes.size > failedBefore);
            if (totalExpired > 0) {
                this.customLoggerService.log(`listing expiry sweep: expired ${totalExpired} stale listing(s)`);
            }
        } finally {
            this.sweepInFlight = false;
        }
    }
}
```

There is no cron expression. Despite the root `CLAUDE.md` saying scheduled work lives in `src/components/cron/` using `@nestjs/schedule`, this scheduler is a plain `setInterval` every five minutes (`SWEEP_INTERVAL_MS = 300000`), registered as a plain provider in `order.module.ts:42`. The class comment explains it copies the lifecycle of the poller's own `ChainEventSourcePoller` (start, stop, in flight guard) "rather than @nestjs/schedule, since this module has no other cron dependency to justify pulling that module in." In practice `@nestjs/schedule` is already a dependency of the app, so the reason is consistency with the poller more than avoiding a package. One practical consequence of `setInterval`: the first sweep happens five minutes after boot, not at boot, so after a long outage the backlog waits five minutes.

The environment gate is `chainConfig.pollerEnabled`, the same `POLLER_ENABLED` flag from the AWS secret that switches the chain poller on and off (file `03` has the table; when the key is missing it falls back to `true` only when `NODE_ENV` trimmed equals `production`, which PM2 sets via `pm2.config.js`). The `onModuleInit` comment gives the reason: "The poller trails the chain, so a sweep running anywhere the poller is not (a developer's machine pointed at the UAT/prod secret) can mark a listing expired before the poller has seen a fill that landed in time." The loader's own comment recalls the 29 September 2026 incident behind the flag, a developer's local backend polling the shared UAT database alongside the UAT server and fighting over the cursor row.

`sweep()` has two safety properties. The `sweepInFlight` boolean means that if one sweep is somehow still running five minutes later (a huge backlog, a slow database), the next tick logs at debug level and returns instead of running two sweeps over the same rows. And the `do ... while` loop keeps pulling batches until a batch "expires nothing and fails nothing new," so a pile of 1000 simultaneously expired listings is cleared in one tick (five batches) rather than five ticks. The "fails nothing new" half matters: a batch whose only row threw would otherwise expire zero, end the loop, and leave every row behind the bad one untouched until the next tick. Because each batch excludes known failures, the loop always terminates: every pass either expires rows (which then no longer match), or adds new hashes to the finite skip set, or ends. The `.catch` inside the interval callback ensures a rejected sweep is logged rather than becoming an unhandled promise rejection that could crash the Node process.

Running more than one instance is safe by construction. `pm2.config.js` defines a single process, but if the API were ever scaled out with `POLLER_ENABLED` true on several machines, two sweeps racing on the same row would both issue the conditional `UPDATE ... WHERE status = 'active'`, and only one can match. The history insert ignores duplicates and the listing status write is an idempotent upsert.

## Domain expiry status utilities

A naming trap to avoid: "expiry" means two completely different things in this module. Everything above is about the order's own Seaport validity window (`endTime`). The two utilities below are about the domain name's real world registration expiry, the date the ENS or similar registration lapses, stored in `tbl_domain_detail_bc.expiryDate`. A seller cares about both: a seven day listing on a name that lapses tomorrow is a trap for the buyer.

```ts
// src/components/marketplacev2/order/util/expiry-status.util.ts
const EXPIRABLE_PROVIDERS_SQL_LIST = `'ENS', 'BinanceSmartChain', 'Arbitrum'`;
const EXPIRING_SOON_WINDOW_SECONDS = 30 * 24 * 60 * 60;
const GRACE_PERIOD_WINDOW_SECONDS = 90 * 24 * 60 * 60;

export function expiryStatusCaseSql(alias: string): string {
    return `(CASE
        WHEN ${alias}."domainProvider" IN (${EXPIRABLE_PROVIDERS_SQL_LIST})
            AND ${alias}."expiryDate" ~ '^[0-9]+$'
        THEN
            CASE
                WHEN ${alias}."expiryDate"::BIGINT < (EXTRACT(EPOCH FROM NOW())::BIGINT - ${GRACE_PERIOD_WINDOW_SECONDS}) THEN 'Expired'
                WHEN ${alias}."expiryDate"::BIGINT < EXTRACT(EPOCH FROM NOW())::BIGINT THEN 'GracePeriod'
                WHEN ${alias}."expiryDate"::BIGINT <= (EXTRACT(EPOCH FROM NOW())::BIGINT + ${EXPIRING_SOON_WINDOW_SECONDS}) THEN 'ExpiringSoon'
                ELSE 'Normal'
            END
        ELSE 'Normal'
    END)`;
}
```

```ts
// src/components/marketplacev2/order/util/domain-expiry-status.util.ts
const EXPIRABLE_PROVIDERS = ['ENS', 'BinanceSmartChain', 'Arbitrum'];
const EXPIRING_SOON_WINDOW_SECONDS = 30 * 24 * 60 * 60;
const GRACE_PERIOD_WINDOW_SECONDS = 90 * 24 * 60 * 60;

export type DomainExpiryStatus = 'Expired' | 'GracePeriod' | 'ExpiringSoon' | 'Normal';

export function classifyDomainExpiry(domainProvider: string, expiryDate: string | null | undefined): DomainExpiryStatus {
    if (!expiryDate || !EXPIRABLE_PROVIDERS.includes(domainProvider) || !/^\d+$/.test(expiryDate)) {
        return 'Normal';
    }
    const expirySeconds = Number(expiryDate);
    const nowSeconds = Math.floor(Date.now() / 1000);
    if (expirySeconds < nowSeconds - GRACE_PERIOD_WINDOW_SECONDS) return 'Expired';
    if (expirySeconds < nowSeconds) return 'GracePeriod';
    if (expirySeconds <= nowSeconds + EXPIRING_SOON_WINDOW_SECONDS) return 'ExpiringSoon';
    return 'Normal';
}
```

Both produce the same four buckets with identical boundaries:

| Bucket | Condition, with `expiryDate` in Unix seconds |
|---|---|
| `Expired` | More than 90 days past expiry. |
| `GracePeriod` | Expired, but within the 90 day grace period (ENS style registries let the old owner renew during this window). |
| `ExpiringSoon` | Not yet expired, and expiring within the next 30 days (inclusive). |
| `Normal` | Everything else, including any provider not in the expirable list, a missing date, or a date that is not purely digits. |

Only three providers carry a real expiry: `ENS`, `BinanceSmartChain` and `Arbitrum`. Unstoppable Domains style names are bought once and never expire, so they are always `Normal`. The digits only test (`~ '^[0-9]+$'` in SQL, `/^\d+$/` in JavaScript) guards the cast: `expiryDate` is a string column, and casting a non numeric string to `BIGINT` in Postgres would make the whole query fail.

There are two copies because the two consumers work differently. `GET /my-listings` filters in Postgres, so it embeds `expiryStatusCaseSql('bc')` as a select column and as the `WHERE` clause for its `expired` and `expiring_soon` categories (`order.service.ts:173` to `174`). `GET /my-domains` already loads every one of the caller's domain rows into memory and paginates in JavaScript, so it calls `classifyDomainExpiry` per row (`order.service.ts:831`). And there is actually a third copy: the SQL version's comment says it "intentionally mirror[s] DomainDetailBCRepo.findAllByUser's identical CASE expression" in `src/components/domain/domain-detail/domain-detail-bc.repo.ts` (lines 17 and 49 to 50), duplicated rather than imported so that this marketplacev2 feature never depends on that file at edit time. The comments ask that all copies be updated together by hand. That is a classic drift risk, softened only by the fact that the thresholds are domain rules that rarely change.

## The specs for this part

`order/order.service.cancel.spec.ts` builds a fixture `OrderEntity` (maker `0x1111...`, hash `0xabab...`, `endTime` `'2000'`) and a mock repository whose transaction runs the callback with a mock `manager.update`. Its six tests: "TC3.1" the maker cancelling their own active order returns exactly `{ orderHash, status: 'cancelled', offChainOnly: true, message }`, calls `update(OrderEntity, { orderHash, status: 'active' }, { status: 'cancelled' })`, calls `upsertUnlisted` with the user id, `example.eth` and `'42'`, records a `CANCELLED` event with `txHash: null`, and the message does not match `/cancelled on chain/i`; "TC3.2" a different wallet gets `ForbiddenException`; "TC3.3" a `filled` order gets `ConflictException` and `update` is never called; "TC3.4" an already `cancelled` order with zero affected rows still returns `cancelled` and writes no listing status or history; "TC3.10" an `invalid` order takes the same zero affected path; "TC3.5" a missing order rejects with "No order found for hash ...".

`order/order.service.expiry-sweep.spec.ts` constructs the service with a mocked `find` and replaces `service.expireIfStale` with a Jest mock, so it tests the batch logic in isolation. Its four tests: the query selects `status` `active`, orders by `{ endTime: 'ASC' }`, takes `200`, and has no `orderHash` filter on a first pass; with three rows where the middle `expireIfStale` returns `false`, the batch returns `2`; when the middle row throws "deadlock detected," the other two still expire, the result is `2`, the skip set contains exactly `0xhash2`, and the error log mentions it; and a pre populated skip set produces a TypeORM `Not` operator (`type` `'not'`) wrapping the excluded hashes.

`order/listing-expiry.scheduler.spec.ts` drives the scheduler with a queue of fake batch results. Its five tests: with `pollerEnabled` false, `onModuleInit` logs a message containing "POLLER_ENABLED is false" and, even after advancing fake timers by an hour, never calls `expireAllStaleBatch`; with it true, advancing fake timers by five minutes triggers a call; batches of 5, 2 and 0 produce exactly three calls and a log containing "expired 7"; a first batch that expires nothing but adds `0xbad` to the skip set keeps the loop going (three calls), and the same `Set` instance is passed to every call; and starting a second `sweep()` while the first is blocked on an unresolved promise results in only one call.

What is not covered: `expireIfStale` itself has no direct test anywhere (the create and parity specs mock its query builder to return "nothing stale"), so the `RETURNING` handling, the legacy `userId` fallback, and the listing status and history writes on expiry are untested. Neither servable function nor either domain expiry classifier has a spec, despite being pure functions that would be trivial to test at their boundaries.

## Bugs, risks and inconsistencies

Soft cancel leaves the order fillable on Seaport (`order.service.ts:761`). As walked through above, anyone holding a signature fetched while the order was active can still buy at the old price until `endTime`, and the response message invites sellers to believe otherwise. Offering an on chain `cancel` from the stored `rawOrder` would close it.

Cancel reports `cancelled` for orders that are not (`order.service.ts:778`). When the conditional update affects zero rows because the order is `expired` or `invalid`, the service still returns `status: 'cancelled'`, and the spec "TC3.10" enshrines that for `invalid`. A frontend that trusts the response will show "cancelled" while `GET /:orderHash` and `my-listings` say `invalid` or `expired`.

Cancel uses the caller's `userId`, not the order's (`order.service.ts:765`). `expireIfStale` carefully prefers the stored `order.userId`; `cancel` writes listing status and history under whoever is logged in. If one wallet is linked to two accounts and the second account cancels, account two gets a spurious `UNLISTED` row and history entry while account one's status row stays `LISTED` forever, which `my-domains` will show.

An active order past its `endTime` can be "cancelled" (`order.service.ts:761`). If the sweep has not reached it yet, cancel's update matches and history records `CANCELLED` for a listing that had actually expired. Minor, but it makes the history table say something untrue.

Only the first returned row is handled in `expireIfStale` (`order.service.ts:570`). The `UPDATE` can in principle affect several rows for one token, but only `rows[0]` gets a history event. This only matters if the partial unique index is missing (see file `03` on the autogenerated migration), but in that world some expirations would be silent.

A legacy row with no resolvable user expires without updating listing status (`order.service.ts:572`). The order flips to `expired` but `tbl_marketplacev2_listing_status` keeps saying `LISTED`, and because the order is no longer `active` no later sweep will revisit it. Only a manual `POST /listing-status/sync` would correct it.

History timestamps are sweep time, not expiry time (`order.service.ts:582`). `occurredAt: new Date()` records when the sweep noticed, up to five minutes late normally, and arbitrarily late in an environment with `POLLER_ENABLED` false, where expiry only ever happens on relist.

With the sweep disabled, the seller's own "active" filter lies (`order.service.ts:168`). `CATEGORY_FILTERS.active` is `o.status = 'active'` only, not `applyServableWhere`, so `GET /my-listings?category=active` shows dead listings as active for up to five minutes normally, and indefinitely wherever the scheduler is gated off. That directly contradicts the servable util's own instruction that "status alone is not enough."

The `expired` and `expiring_soon` listing categories can never match today (`expiry-status.util.ts:16`). The only listable contract is `DOMAIN_NFT_ADDRESS_POLYGON_UD`, Unstoppable Domains on Polygon, and none of that contract's providers is in the expirable list of `ENS`, `BinanceSmartChain` and `Arbitrum`. So every row in `my-listings` is `Normal`, and those two filter tabs always come back empty. They will only become meaningful if an expiring registry's NFT is ever made listable.

Shutdown hooks are not enabled. `main.ts` never calls `app.enableShutdownHooks()`, so `onModuleDestroy`, and with it `stop()`, never runs on a PM2 restart's `SIGTERM`; it only runs when something calls `app.close()`, as the specs do. The process exits anyway, so the interval dies with it, but a sweep in the middle of a transaction is cut off rather than allowed to finish (each row is its own transaction, so at worst one row's work is rolled back).

The scheduler relies on the Node clock, and Seaport relies on block time (`servable-order.util.ts:19`). Polygon block timestamps can drift a few seconds from wall clock time, so for a few seconds either side of `endTime` the backend and Seaport can disagree about whether an order is live. This is unavoidable and harmless, but it is why a buyer clicking "buy" in the last seconds of a listing can see a revert.

## Frontend note

Two practical takeaways for building the seller and buyer screens. On the seller side, present cancel as "remove from Endless Domains," not "revoke," and consider offering the on chain cancel for sellers who are relisting at a higher price, since that is exactly the case where an old signature in someone's browser tab costs the seller money. On the buyer side, never trust a status you fetched a while ago: the servable rule is "active and not past `endTime`," and since that depends on the clock, recompute it locally before enabling the buy button, then let Seaport's own `getOrderStatus` and the transaction itself have the final word.
