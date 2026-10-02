# 07. Watchlist, Listing Status, Listing History, and Transaction Records

## Four tables around one orders table

`tbl_marketplacev2_orders` is the source of truth for every v2 listing, and `06` showed how it is browsed. This file is about the four smaller tables that orbit it, each of which exists to answer one specific question the orders table answers badly or not at all. The watchlist answers "which domains has this user starred, regardless of how many times they have been relisted?". The listing status table answers "for this user, is each of their domains listed right now?", in a way that can be looked up by `userId` instead of by wallet address. The listing history table answers "what happened to this user's listings, and when?", as a permanent append only log. And the transaction table answers "what sales actually settled on chain, for how much, in which block, at what gas cost?", as the financial settlement record.

The commits that introduced these are worth knowing by name if you go digging in `git log`: the listing history read API is `225b5c86`, the transaction record table is `de4d906e`, and the transaction history routes are `008318b0`. The watchlist and listing status pieces arrived with the broader v2 order work before and around `64451cb4`.

Here is the whole map at a glance before we go table by table.

| Table | Entity file | Written by | Read by | Shape |
|---|---|---|---|---|
| `tbl_marketplacev2_watchlist` | `order/entity/watchlist.entity.ts` | `POST /watchlist` and `DELETE /watchlist/:id` (API) | `GET /watchlist` | one row per user per token, insert and delete |
| `tbl_marketplacev2_listing_status` | `listing-status/entity/listing-status.entity.ts` | order create and cancel, expiry sweep, poller fill, cancel, and invalidate, plus the sync endpoint | `GET /my-domains` | one row per user, domain, token, overwritten in place |
| `tbl_marketplacev2_listing_history` | `listing-status/entity/listing-history.entity.ts` | same sites as listing status, never the sync endpoint | `GET /listing-history`, the SOLD rule in `GET /my-domains` | append only event log |
| `tbl_marketplacev2_transactions` | `transaction/entity/transaction.entity.ts` | the poller only, inside the fill transaction | `GET /transactions/recent`, `GET /transactions/mine`, `GET /transactions/:orderHash` | append only settlement record |

All routes below are on `OrderController` (`src/components/marketplacev2/order/order.controller.ts`) under the prefix `/api/v1/marketplacev2/orders`. None of these tables has a hand written migration in the repo; like the rest of the codebase they are created by the TypeORM migrations the deploy script generates from the entities (see the infrastructure notes), which is why the entity decorators are the authoritative schema.

## The watchlist

### Why a brand new table

```ts
// src/components/marketplacev2/order/entity/watchlist.entity.ts
@Entity({ name: 'tbl_marketplacev2_watchlist' })
@Unique('uq_marketplacev2_watchlist_user_token_contract_token_id', ['userId', 'tokenContract', 'tokenId'])
@Index('idx_marketplacev2_watchlist_user_id', ['userId'])
export class WatchlistEntity {
    @PrimaryGeneratedColumn('uuid')
    public id: string;

    /** req.user.userId, not walletAddress - a login-only preference, no wallet check. */
    @Column({ type: 'varchar' })
    public userId: string;

    /** Checksummed, same convention as OrderEntity.maker/tokenContract. */
    @Column({ type: 'varchar' })
    public tokenContract: string;

    /** Matches OrderEntity's own convention - a numeric string, never a JS number. */
    @Column({ type: 'varchar' })
    public tokenId: string;

    @Column({ type: 'timestamp' })
    public createdAt: Date;
}
```

The entity's doc comment explains a real design lesson from v1. The old `tbl_watchlist` stored a `domain_id` that was actually `DomainListing.id`, a per listing UUID. When a seller cancelled and relisted, the new listing got a new id, and every user who had watched the old one was now watching a dead row. v2 keys the watchlist on `(tokenContract, tokenId)`, which is the NFT's own permanent identity and never changes no matter how many times it is listed, sold, or relisted. It deliberately does not key on `domainName` either, because names are display strings that can differ in casing between sources while the token id cannot.

Column by column: `id` is a generated UUID primary key, and it is what the frontend uses to delete a row. `userId` is the logged in user's internal id from the JWT, not a wallet, because starring a domain is a UI preference, not an on chain action, and should not require wallet verification. `tokenContract` is the NFT contract address, normalised to its EIP 55 checksummed form before insert. `tokenId` is a decimal string because ERC 721 token ids routinely exceed `Number.MAX_SAFE_INTEGER`. `createdAt` is set by the service, not by a `@CreateDateColumn`. The unique constraint on `(userId, tokenContract, tokenId)` is what makes "add twice" a no op, and the separate index on `userId` serves the list query.

### Endpoints and DTOs

| Method and path | Guards | Body or params | Returns |
|---|---|---|---|
| `POST /watchlist` | `AccessTokenGuard` | `WatchlistAddDto { tokenContract, tokenId }` | the created or already existing row (`WatchlistAddResult`) |
| `DELETE /watchlist/:id` | `AccessTokenGuard` | `id` path param | `result: null` with message `Removed from watchlist` |
| `GET /watchlist` | `AccessTokenGuard` | none | `WatchlistItemDto[]`, not paginated |

```ts
// src/components/marketplacev2/order/dto/watchlist-add.dto.ts
export class WatchlistAddDto {
    @Matches(ETH_ADDRESS_PATTERN)
    tokenContract: string;

    /** Matches OrderEntity's own convention - a numeric string, never a JS number. */
    @IsNumberString()
    tokenId: string;
}
```

```ts
// src/components/marketplacev2/order/dto/watchlist-item.dto.ts
export interface WatchlistItemDto {
    id: string;
    tokenContract: string;
    tokenId: string;
    createdAt: Date;
    live: boolean;
    order: OrderSummaryDto | null;
}
```

`tokenContract` must match `ETH_ADDRESS_PATTERN` (a `0x` plus forty hex characters regex from `dto/eth-format.constants.ts`), and `tokenId` must be a numeric string. `WatchlistAddResult` from `interface/watchlist-service.interface.ts` is just `{ id, tokenContract, tokenId, createdAt }`. Each `WatchlistItemDto` carries the row itself plus `order`, the most recent order ever made for that token in the same `OrderSummaryDto` shape browse uses (no signature), and `live`, a boolean that is true only when that order is currently servable. `order` is `null` only when no order has ever existed for the token; `live` is never omitted.

### The service, method by method

```ts
// src/components/marketplacev2/order/watchlist.service.ts
async add(userId: string, tokenContract: string, tokenId: string): Promise<WatchlistAddResult> {
    const normalizedContract = getAddress(tokenContract);
    try {
        const row = this.watchlistRepository.create({ userId, tokenContract: normalizedContract, tokenId, createdAt: new Date() });
        return await this.watchlistRepository.save(row);
    } catch (err) {
        if ((err as { code?: string })?.code === PostgresErrorCode.UNIQUE_VIOLATION) {
            return await this.watchlistRepository.findOneOrFail({ where: { userId, tokenContract: normalizedContract, tokenId } });
        }
        throw err;
    }
}
```

`add` is idempotent through the database rather than through a "check then insert" read. It just tries to insert; if Postgres raises unique violation `23505`, the row already exists, so it fetches and returns that existing row with a 200. This is the right pattern because a read then insert has a race (two quick double clicks both read "not there" and both insert), while the unique constraint makes the database the referee. `getAddress` from ethers v6 both validates and checksums the address, so `0xabc...` and `0xABC...` collapse to one canonical form before they can produce two rows. One caveat worth knowing: `getAddress` throws for an address whose mixed casing has an invalid checksum, and that throw happens before the `try`, so a user submitting a badly cased address gets a 500, not a 400.

```ts
// src/components/marketplacev2/order/watchlist.service.ts
async remove(userId: string, id: string): Promise<void> {
    const result = await this.watchlistRepository.delete({ id, userId });
    if (!result.affected) {
        throw new NotFoundException(`No watchlist entry found for id ${id}.`);
    }
}
```

`remove` scopes the delete by both `id` and `userId` in a single statement. If the id belongs to someone else, zero rows are affected, exactly the same as if the id does not exist at all, and both return the same 404. That is a small but real privacy property: an attacker probing ids cannot learn which ones exist for other users, which a "fetch, then compare owner, then 403" design would leak.

```ts
// src/components/marketplacev2/order/watchlist.service.ts
async list(userId: string): Promise<WatchlistItemDto[]> {
    const rows = await this.watchlistRepository.find({ where: { userId }, order: { createdAt: 'DESC' } });
    if (rows.length === 0) {
        return [];
    }

    const tokenContracts = [...new Set(rows.map((row) => row.tokenContract))];
    const tokenIds = [...new Set(rows.map((row) => row.tokenId))];
    const candidateOrders = await this.orderRepository.find({
        where: { tokenContract: In(tokenContracts), tokenId: In(tokenIds) },
        order: { createdAt: 'DESC' }
    });

    const latestByKey = new Map<string, OrderEntity>();
    for (const order of candidateOrders) {
        const key = `${order.tokenContract}::${order.tokenId}`;
        if (!latestByKey.has(key)) {
            latestByKey.set(key, order);
        }
    }

    return rows.map((row) => {
        const order = latestByKey.get(`${row.tokenContract}::${row.tokenId}`) ?? null;
        return {
            id: row.id,
            tokenContract: row.tokenContract,
            tokenId: row.tokenId,
            createdAt: row.createdAt,
            live: order ? isOrderServable(order) : false,
            order: order ? toOrderSummary(order) : null
        };
    });
}
```

`list` is a nice example of avoiding an N+1 query. The naive approach would loop over watchlist rows and run one "latest order for this token" query per row, so 50 watched domains would cost 51 queries. Instead it runs exactly two: one for the user's watchlist rows (served by `idx_marketplacev2_watchlist_user_id`), and one for every order matching any watched contract and any watched token id, newest first (served by `idx_marketplacev2_orders_token_id`). Because the rows come back sorted newest first, the first order seen for each `contract::tokenId` key is the latest one, and later ones are ignored. The comment is honest that `IN(contracts) AND IN(tokenIds)` is a superset when more than one contract is watched (it would match contract A with a token id that is only watched on contract B), but the exact pair lookup in the map discards those extras, so the result is correct either way.

The SQL it produces is approximately:

```ts
// approximate SQL generated by WatchlistService.list (illustrative)
SELECT * FROM tbl_marketplacev2_watchlist WHERE "userId" = $1 ORDER BY "createdAt" DESC;
SELECT * FROM tbl_marketplacev2_orders
WHERE "tokenContract" IN ($1) AND "tokenId" IN ($2, $3, ...)
ORDER BY "createdAt" DESC;
```

`live` is computed with `isOrderServable`, the in memory twin of the SQL `applyServableWhere` predicate described in `06`, so "live on the watchlist" and "visible on browse" can never disagree. Because the join picks the latest order regardless of status, a domain that sold and was never relisted still appears, with `live: false` and its terminal `filled` order attached, rather than silently vanishing from the user's list.

## Listing status: the "is it listed right now" cache

### The entity and the enum

```ts
// src/components/marketplacev2/listing-status/entity/listing-status.entity.ts
@Entity({ name: 'tbl_marketplacev2_listing_status' })
@Unique('uq_marketplacev2_listing_status_user_domain_token', ['userId', 'domainName', 'tokenId'])
@Index('idx_marketplacev2_listing_status_user_id', ['userId'])
export class ListingStatusEntity {
    @PrimaryGeneratedColumn('uuid')
    public id: string;

    @Column({ type: 'varchar' })
    public userId: string;

    @Column({ type: 'varchar' })
    public domainName: string;

    @Column({ type: 'varchar' })
    public tokenId: string;

    @Column({ type: 'enum', enum: ListingStatus })
    public status: ListingStatus;

    @Column({ type: 'timestamp' })
    public updatedAt: Date;
}
```

```ts
// src/components/marketplacev2/listing-status/enum/listing-status.enum.ts
export enum ListingStatus {
    LISTED = 'LISTED',
    UNLISTED = 'UNLISTED'
}
```

The doc comment calls this a current state cache only, never a source of truth, and it is worth understanding why it exists at all. `tbl_marketplacev2_orders` only stores the seller as a wallet address (`maker`), but "my domains" starts from a `userId`. Without this table, my domains would need to look up every wallet the user has, then join orders by maker, then work out per token which order is the current one. With it, my domains is one indexed `WHERE userId = ?` read. The price is that this is denormalised data that must be kept in step with the orders table at every write site, which is exactly what the services below are designed to guarantee. Browse, stats, the watchlist, and order validity never read it.

Columns: `id` UUID; `userId` the seller's internal id; `domainName` and `tokenId` identify the domain (note: not `tokenContract`, which is implied, since only the one configured domain contract is ever listed); `status` the Postgres enum `LISTED` or `UNLISTED`; `updatedAt` stamped on every write. The unique constraint `(userId, domainName, tokenId)` is both the integrity rule and the conflict target for upserts. Absence of a row means UNLISTED, a rule both the read side and the sync side follow.

### `ListingStatusService`

```ts
// src/components/marketplacev2/listing-status/listing-status.service.ts
async findAllForUser(userId: string): Promise<ListingStatusEntity[]> {
    return this.listingStatusRepository.find({ where: { userId } });
}

async upsertListed(manager: EntityManager, userId: string, domainName: string, tokenId: string): Promise<void> {
    await this.upsert(manager, userId, domainName, tokenId, ListingStatus.LISTED);
}

async upsertUnlisted(manager: EntityManager, userId: string, domainName: string, tokenId: string): Promise<void> {
    await this.upsert(manager, userId, domainName, tokenId, ListingStatus.UNLISTED);
}

private async upsert(manager: EntityManager, userId: string, domainName: string, tokenId: string, status: ListingStatus): Promise<void> {
    await manager.getRepository(ListingStatusEntity).upsert({ userId, domainName, tokenId, status, updatedAt: new Date() }, ['userId', 'domainName', 'tokenId']);
}
```

The most important design point here is the `manager: EntityManager` parameter. Every write method takes the caller's transaction manager instead of using the injected repository, because each write must land in the same database transaction as the orders table change that caused it. If the order insert commits but the status upsert fails, or vice versa, the two tables would disagree, and the whole value of the cache is that it never disagrees. Passing the manager through is how you compose transactional writes across services in TypeORM; using `this.listingStatusRepository` inside a `manager.transaction(...)` callback would silently run on a different connection, outside the transaction. TypeORM's `upsert(values, conflictPaths)` compiles to `INSERT ... ON CONFLICT ("userId", "domainName", "tokenId") DO UPDATE SET ...`, so the first write for a domain inserts and every later write overwrites in place.

### `ListingStatusModule`

```ts
// src/components/marketplacev2/listing-status/listing-status.module.ts
@Module({
    imports: [LoggerModule, TypeOrmModule.forFeature([ListingStatusEntity, ListingHistoryEntity])],
    providers: [ListingStatusService, ListingHistoryService],
    exports: [ListingStatusService, ListingHistoryService]
})
export class ListingStatusModule {}
```

Both `OrderModule` and `PollerModule` need to write these tables, and those two are siblings, so this module sits underneath both rather than living inside either one. The comment notes this avoids a real circular import, since `OrderModule` already imports `DomainDetailModule`. Note that these two services are provided by class, not by an interface token string, which differs from the project's usual `@Inject('SomethingInterface')` convention.

## The manual sync endpoint: `POST /listing-status/sync`

### What it does

```ts
// src/components/marketplacev2/order/order.controller.ts
@ApiBearerAuth('defaultBearerAuth')
@Post('listing-status/sync')
@UseGuards(AccessTokenGuard)
async syncListingStatus(@Req() req: Request): Promise<Response> {
    return new Response('Listing status synced', await this.listingStatusSyncService.sync(req.user['userId']));
}
```

```ts
// src/components/marketplacev2/order/listing-status-sync.service.ts
async sync(userId: string): Promise<ListingStatusSyncResult> {
    await this.domainDetailService.refreshDomainDetailData(userId);
    const domains = await this.domainDetailBCRepo.findById(userId);
    const currentlyOwnedTokenIds = domains.map((domain) => domain.token_id).filter((tokenId): tokenId is string => Boolean(tokenId));
    return this.listingStatusService.syncAfterRefresh(userId, currentlyOwnedTokenIds);
}
```

```ts
// src/components/marketplacev2/listing-status/listing-status.service.ts
async syncAfterRefresh(userId: string, currentlyOwnedTokenIds: string[]): Promise<{ correctedCount: number }> {
    const ownedTokenIds = new Set(currentlyOwnedTokenIds);
    const listedRows = await this.listingStatusRepository.find({ where: { userId, status: ListingStatus.LISTED } });
    const staleRows = listedRows.filter((row) => !ownedTokenIds.has(row.tokenId));
    if (staleRows.length === 0) {
        return { correctedCount: 0 };
    }

    await this.listingStatusRepository.update({ userId, tokenId: In(staleRows.map((row) => row.tokenId)), status: ListingStatus.LISTED }, { status: ListingStatus.UNLISTED, updatedAt: new Date() });
    return { correctedCount: staleRows.length };
}
```

The response is `{ correctedCount: number }` (`interface/listing-status-sync.service.interface.ts`), the number of the caller's LISTED rows flipped to UNLISTED, with 0 meaning nothing was stale. The flow is three steps.

1. Refresh the user's ownership from the chain providers by calling the same `DomainDetailService.refreshDomainDetailData(userId)` that powers `GET /api/v1/domain/detail/refresh_domain`. That method fetches every NFT the user's EVM wallet owns through six Alchemy backed services (Freename, Arbitrum, BNB, UD Base, ENS, UD) and the Solana wallet through Moralis, then rewrites the user's rows in `tbl_domain_detail_bc`.
2. Re read the full owned set with `findById`, rather than trusting the refresh's own return value, because (per the comment) that return value is paginated to 100 and would under scan a user who owns more.
3. Flip every LISTED status row whose `tokenId` is no longer owned to UNLISTED. No rows are inserted, since absence already means UNLISTED.

The use case is a seller who listed a domain and then transferred it out of their wallet by some other means. The poller normally catches that (a `Transfer` event invalidates the order and flips status, see the poller file), so the comments are clear this is "a secondary convenience only", a button for a user who sees something stale and wants it fixed now. It is guarded by `AccessTokenGuard` only, no wallet verification, because it corrects the caller's own rows by `userId` and performs no chain write.

### Why it is rate limited in `main.ts`

```ts
// src/main.ts
const apiCallLimiter = rateLimit({
    windowMs: Number(secrets.API_CALL_LIMIT_RESET_TIME) * 60 * 1000,
    max: Number(secrets.API_CALL_LIMIT),
    message: {
        statusCode: HttpStatus.TOO_MANY_REQUESTS,
        message: ['Too many request for API call from this IP, please try again after ' + secrets.API_CALL_LIMIT_RESET_TIME + ' minutes'],
        error: 'Too Many Requests'
    }
});
app.use('/api/v1/auth/forgot-password', apiCallLimiter);
app.use('/api/v1/domain/detail/refresh_domain', apiCallLimiter);
app.use('/api/v1/marketplacev2/orders/listing-status/sync', apiCallLimiter);
```

Every call to sync triggers the full multi provider refresh underneath, which means six or more paid third party API calls (Alchemy compute units and Moralis credits), several of which paginate through every NFT the wallet holds, plus a delete and reinsert of the user's domain detail rows. `refresh_domain` was already rate limited for exactly this reason. If sync were left unlimited, it would become a second, unguarded door to the same expensive operation, and anyone with a valid JWT could burn through the provider quota or hammer the database with a simple loop. The controller comment says precisely this: it "must not become a second, unguarded path to it". The window and maximum come from Secrets Manager (`API_CALL_LIMIT_RESET_TIME` in minutes, `API_CALL_LIMIT` requests), and `expressApp.set('trust proxy', 1)` earlier in `main.ts` makes `req.ip` the real client address behind the load balancer rather than the balancer's own.

There is a subtlety in how this is wired that the comment's phrase "shares that same route's rate limiter" undersells. `express-rate-limit` keeps one counter store per limiter instance, keyed by default on `req.ip`. Because the same `apiCallLimiter` instance is mounted on all three paths, a single IP has one shared budget across forgot password, refresh domain, and listing status sync combined. A user who clicks sync a few times can exhaust their own refresh budget and their forgot password budget, and conversely, an office or mobile carrier NAT where many users share one IP shares one budget for all three. The default store is also in process memory, so with several PM2 workers or several EC2 instances behind the balancer each process counts separately, and the effective limit is the configured limit times the number of processes. Separate limiter instances per path, keyed on `userId` for the authenticated two, and a shared store such as Redis would make the limit mean what it says.

## Listing history: the append only event log

### The entity and the event types

```ts
// src/components/marketplacev2/listing-status/entity/listing-history.entity.ts
@Entity({ name: 'tbl_marketplacev2_listing_history' })
@Unique('uq_marketplacev2_listing_history_order_hash_event_type', ['orderHash', 'eventType'])
@Index('idx_marketplacev2_listing_history_user_id_created_at', ['userId', 'createdAt'])
@Index('idx_marketplacev2_listing_history_token_contract_token_id_created_at', ['tokenContract', 'tokenId', 'createdAt'])
export class ListingHistoryEntity {
    @PrimaryGeneratedColumn('uuid')
    public id: string;

    @Column({ type: 'varchar' })
    public userId: string;

    @Column({ type: 'varchar' })
    public domainName: string;

    @Column({ type: 'varchar' })
    public tokenId: string;

    @Column({ type: 'varchar' })
    public tokenContract: string;

    @Column({ type: 'varchar', length: 66 })
    public orderHash: string;

    @Column({ type: 'enum', enum: ListingEventType })
    public eventType: ListingEventType;

    @Column({ type: 'varchar', nullable: true })
    public txHash: string | null;

    @Column({ type: 'timestamp' })
    public occurredAt: Date;

    @Column({ type: 'timestamp' })
    public createdAt: Date;
}
```

```ts
// src/components/marketplacev2/listing-status/enum/listing-event-type.enum.ts
export enum ListingEventType {
    LISTED = 'LISTED',
    CANCELLED = 'CANCELLED',
    SOLD = 'SOLD',
    INVALIDATED = 'INVALIDATED',
    /** The listing's own endTime passed while it was still active - see OrderService.expireIfStale. */
    EXPIRED = 'EXPIRED'
}
```

| Column | Type | Meaning |
|---|---|---|
| `id` | `uuid` PK | row id, never returned to clients |
| `userId` | `varchar` | the seller's internal id; the table has no wallet column at all |
| `domainName` | `varchar` | display name at the time of the event |
| `tokenId` | `varchar` | NFT token id as a decimal string |
| `tokenContract` | `varchar` | NFT contract address |
| `orderHash` | `varchar(66)` | the order this event belongs to |
| `eventType` | enum | `LISTED`, `CANCELLED`, `SOLD`, `INVALIDATED`, `EXPIRED` |
| `txHash` | `varchar`, nullable | set for on chain events (SOLD, INVALIDATED, on chain CANCELLED); `null` for off chain ones (LISTED, off chain CANCELLED, EXPIRED) |
| `occurredAt` | `timestamp` | when it really happened: block time for chain events, request time for API events |
| `createdAt` | `timestamp` | when the row was written, for debugging only |

The split between `occurredAt` and `createdAt` is a classic event sourcing habit. If the poller is down for an hour and then catches up, a sale that happened at 10:00 is written at 11:00; `occurredAt` says 10:00, `createdAt` says 11:00, and any UI should render `occurredAt`. The enum's comment explains why SOLD and INVALIDATED exist alongside the user triggered LISTED and CANCELLED: recording only what the user did would leave a domain that quietly sold or was invalidated with no explanation of why it stopped being listed. EXPIRED was added for listings whose `endTime` passed while still active.

The unique constraint on `(orderHash, eventType)` is the idempotency mechanism. An order can be created once, cancelled once, sold once, invalidated once, and expired once, so one row per pair is always correct, and a replay of the same event becomes a no op at the database.

### Who writes it, and in what transaction

Every write site pairs a listing history `record` with a listing status upsert, inside the same transaction as the orders table update. This table is the most useful single place to see who changes order state in v2.

| Event | Writer | File | Trigger | `txHash` | `occurredAt` |
|---|---|---|---|---|---|
| `LISTED` | API | `order/order.service.ts` `create()` | `POST /marketplacev2/orders` | `null` | `new Date()` at request time |
| `CANCELLED` | API | `order/order.service.ts` `cancel()` | `POST /:orderHash/cancel` (off chain soft cancel) | `null` | `new Date()` |
| `EXPIRED` | API process (scheduler and create path) | `order/order.service.ts` `expireIfStale()` | `ListingExpiryScheduler` sweep, or a relist attempt finding a stale active row | `null` | `new Date()` at sweep time |
| `SOLD` | poller | `poller/application/seaport-event-application.service.ts` `applyOrderFulfilled` | Seaport `OrderFulfilled` after confirmation depth | fill tx hash | block timestamp |
| `CANCELLED` | poller | same file, `applyOrderCancelled` | Seaport `OrderCancelled` (on chain cancel) | cancel tx hash | block timestamp |
| `INVALIDATED` | poller | `poller/application/domain-event-application.service.ts` | ERC 721 `Transfer` away from the maker (owner changed) | transfer tx hash | block timestamp |
| `INVALIDATED` | poller | same file | `ApprovalForAll` revoked for Seaport (approval revoked) | approval tx hash | block timestamp |

Here is the API side shape, from `create()`:

```ts
// src/components/marketplacev2/order/order.service.ts
await this.orderRepository.manager.transaction(async (manager) => {
    await manager.insert(OrderEntity, entity);
    await this.listingStatusService.upsertListed(manager, userId, dto.domainName, result.tokenId);
    await this.listingHistoryService.record(manager, {
        userId,
        domainName: dto.domainName,
        tokenId: result.tokenId,
        tokenContract: result.tokenContract,
        orderHash: result.orderHash,
        eventType: ListingEventType.LISTED,
        txHash: null,
        occurredAt: new Date()
    });
});
```

And the poller side shape, from the fill handler, which additionally writes the transaction record covered later:

```ts
// src/components/marketplacev2/poller/application/seaport-event-application.service.ts
const userId = await this.resolveOrderUserId(order, `OrderFulfilled for ${orderHash}`);
if (userId) {
    await this.listingStatusService.upsertUnlisted(manager, userId, order.domainName, order.tokenId);
    await this.listingHistoryService.record(manager, {
        userId,
        domainName: order.domainName,
        tokenId: order.tokenId,
        tokenContract: order.tokenContract,
        orderHash,
        eventType: ListingEventType.SOLD,
        txHash: log.transactionHash,
        occurredAt: filledAt
    });
}
```

`resolveOrderUserId` returns `order.userId` if the order row has it (every order created since that column was added does), otherwise falls back to an exact, case sensitive wallet lookup for legacy rows. If that lookup finds nothing, the status and history writes are skipped entirely, deliberately as a plain `if`, not a thrown error, so a missing mapping can never roll back the authoritative order update. The flip side is that for such a legacy order, the event is simply never recorded in history, with only a log line to show for it.

### `record()` and why `.orIgnore()` matters

```ts
// src/components/marketplacev2/listing-status/listing-history.service.ts
async record(manager: EntityManager, event: ListingHistoryEvent): Promise<void> {
    await manager.createQueryBuilder().insert().into(ListingHistoryEntity).values({ ...event, createdAt: new Date() }).orIgnore().execute();
}
```

`.orIgnore()` compiles to `INSERT ... ON CONFLICT DO NOTHING`. The doc comment explains a genuinely important Postgres behaviour that trips up many developers: if you run a plain `INSERT` inside a transaction, it hits a unique violation, and you catch the error in JavaScript and carry on, Postgres has already marked the whole transaction as aborted. Every later statement fails with "current transaction is aborted", and if you reach `COMMIT`, Postgres quietly performs a `ROLLBACK` instead. So the try and catch "worked" in JS, but the order update you thought you made alongside it is gone, and the client got a 200. `ON CONFLICT DO NOTHING` avoids raising the error in the first place, so the transaction stays healthy and the duplicate is a true no op. If you remember one Postgres lesson from this module, make it this one. (The same reasoning is behind `create()` using `manager.insert` instead of `save`, explained in the order creation file.)

### Reading it: `findForUser`, `findSoldForUser`, and `GET /listing-history`

```ts
// src/components/marketplacev2/listing-status/listing-history.service.ts
async findSoldForUser(userId: string): Promise<Map<string, Date>> {
    const rows = await this.listingHistoryRepository.find({ where: { userId, eventType: ListingEventType.SOLD } });
    const latestByKey = new Map<string, Date>();
    for (const row of rows) {
        const key = `${row.domainName}::${row.tokenId}`;
        const existing = latestByKey.get(key);
        if (!existing || row.occurredAt > existing) {
            latestByKey.set(key, row.occurredAt);
        }
    }
    return latestByKey;
}

async findForUser(userId: string, filters: { domainName?: string; tokenId?: string }, skip: number, limit: number): Promise<{ data: ListingHistoryEntity[]; total: number }> {
    const where: FindOptionsWhere<ListingHistoryEntity> = { userId };
    if (filters.domainName) {
        where.domainName = filters.domainName;
    }
    if (filters.tokenId) {
        where.tokenId = filters.tokenId;
    }
    const [data, total] = await this.listingHistoryRepository.findAndCount({ where, order: { occurredAt: 'DESC' }, skip, take: limit });
    return { data, total };
}
```

`findSoldForUser` feeds the SOLD rule in my domains, explained fully in `06`: a domain shows SOLD only if its latest SOLD `occurredAt` is newer than the domain's last ownership refresh. `findForUser` backs the user facing endpoint:

| Method and path | Guards | Query | Returns |
|---|---|---|---|
| `GET /listing-history` | `AccessTokenGuard`, `RequireVerifiedWalletGuard` | `ListingHistoryQueryDto`: `domainName?` (`@IsString`), `tokenId?` (`@IsString`), `page?`, `limit?` (`@Type(() => Number) @IsInt @Min(1)`, defaults 1 and 10) | `paginateResponse` of `ListingHistoryEntryDto[]` |

```ts
// src/components/marketplacev2/order/dto/listing-history-entry.dto.ts
export interface ListingHistoryEntryDto {
    orderHash: string;
    domainName: string;
    tokenId: string;
    tokenContract: string;
    eventType: ListingEventType;
    txHash: string | null;
    occurredAt: Date;
}
```

`OrderService.listingHistory` maps entities to this DTO, deliberately dropping `id`, `userId` (the caller knows who they are), and `createdAt` (audit only). The filters combine with AND, so `?domainName=fox.eth&tokenId=42` narrows to that one domain. It uses `findAndCount`, which runs the page query plus a `COUNT(*)` in one call, ordered newest first. The controller comment admits that `RequireVerifiedWalletGuard` is applied "for parity" with my listings and my purchases even though the query is scoped purely by `userId`, so a user must have signed a wallet challenge within the last hour to read their own history.

A React timeline renders each entry with `occurredAt` and an icon per `eventType`, and links to a block explorer only when `txHash` is not `null`. For one domain's history, pass both `domainName` and `tokenId` from the my domains row.

## The transaction record: `tbl_marketplacev2_transactions`

### Why a separate settlement table

The entity's doc comment draws a sharp contrast with v1, and it is worth reading slowly. In v1, `tbl_domain_buy` (the `BuyDomainListing` entity from `03-commerce-and-marketplace/03`) inserts a row the moment the frontend claims a purchase, in a `Pending` state, with `retryCount` and `reconciliationStatus` columns so a cron can later check the chain and decide whether it really happened. v2 inverts that. Nothing is inserted until the poller has already seen the `OrderFulfilled` event on chain and it has passed the confirmation depth (`CONFIRMATIONS`, see the poller file), so the row is born final. There is no status column, no retry counter, no reconciliation, because there is nothing to reconcile. It is append only: never updated, never deleted. And it is kept separate from listing history because the two serve different audiences: history is a seller's private timeline, including non financial events; this is the financial record, readable by the buyer, the seller, and the public feed.

### Column by column

```ts
// src/components/marketplacev2/transaction/entity/transaction.entity.ts
@Entity({ name: 'tbl_marketplacev2_transactions' })
@Unique('uq_marketplacev2_transactions_order_hash_type', ['orderHash', 'transactionType'])
@Index('idx_marketplacev2_transactions_buyer_occurred_at', ['buyer', 'occurredAt'])
@Index('idx_marketplacev2_transactions_seller_occurred_at', ['seller', 'occurredAt'])
@Index('idx_marketplacev2_transactions_token_contract_token_id', ['tokenContract', 'tokenId'])
@Index('idx_marketplacev2_transactions_occurred_at', ['occurredAt'])
export class TransactionEntity {
    @PrimaryGeneratedColumn('uuid') public id: string;
    @Column({ type: 'enum', enum: TransactionType }) public transactionType: TransactionType;
    @Column({ type: 'varchar', length: 66 }) public orderHash: string;
    @Column({ type: 'varchar', length: 66 }) public txHash: string;
    @Column({ type: 'bigint', transformer: bigintStringTransformer }) public blockNumber: string;
    @Column({ type: 'timestamp' }) public occurredAt: Date;
    @Column({ type: 'int' }) public chainId: number;
    @Column({ type: 'varchar' }) public tokenContract: string;
    @Column({ type: 'varchar' }) public tokenId: string;
    @Column({ type: 'varchar' }) public domainName: string;
    @Column({ type: 'varchar' }) public tld: string;
    @Column({ type: 'varchar' }) public buyer: string;
    @Column({ type: 'varchar' }) public seller: string;
    @Column({ type: 'bigint', transformer: bigintStringTransformer }) public priceUsdt: string;
    @Column({ type: 'bigint', transformer: bigintStringTransformer }) public feeUsdt: string;
    @Column({ type: 'bigint', transformer: bigintStringTransformer }) public sellerUsdt: string;
    @Column({ type: 'bigint', transformer: bigintStringTransformer, nullable: true }) public gasUsed: string | null;
    @Column({ type: 'bigint', transformer: bigintStringTransformer, nullable: true }) public effectiveGasPrice: string | null;
    @Column({ type: 'timestamp' }) public createdAt: Date;
}
```

(Condensed to one line per column here; the real file has a doc comment on most of them.)

| Column | Type | Source at write time | Meaning |
|---|---|---|---|
| `id` | `uuid` PK | generated | internal row id, never returned |
| `transactionType` | enum `SALE` | constant | the kind of settlement; only `SALE` exists today |
| `orderHash` | `varchar(66)` | order row | the order this sale settled |
| `txHash` | `varchar(66)` | `log.transactionHash` | the fill transaction on Polygon |
| `blockNumber` | `bigint` as string | `log.blockNumber` | block containing the fill |
| `occurredAt` | `timestamp` | `log.blockTimestamp * 1000` | block time of the sale |
| `chainId` | `int` | `MARKETPLACEV2_CHAIN_ID` | fixed today, a column so a second chain is a data change |
| `tokenContract` | `varchar` | order row | NFT contract |
| `tokenId` | `varchar` | order row | NFT token id |
| `domainName` | `varchar` | order row | denormalised for display |
| `tld` | `varchar` | order row | denormalised for display |
| `buyer` | `varchar` | `OrderFulfilled.recipient` | who received the NFT |
| `seller` | `varchar` | `order.maker` | who listed it |
| `priceUsdt` | `bigint` as string | order row | total paid, USDT minor units (6 decimals) |
| `feeUsdt` | `bigint` as string | order row | marketplace fee |
| `sellerUsdt` | `bigint` as string | order row | seller proceeds |
| `gasUsed` | `bigint` as string, nullable | `receipt.gasUsed` | gas units consumed by the fill tx |
| `effectiveGasPrice` | `bigint` as string, nullable | `receipt.gasPrice` | wei per gas unit actually paid |
| `createdAt` | `timestamp` | `new Date()` | when the row was written, audit only |

The `bigintStringTransformer` is a pass through that makes it explicit that every amount and block number stays a decimal string, never round tripping through a JS `number` where values above 2^53 would lose precision. Gas cost in wei, if a UI wants it, is `BigInt(gasUsed) * BigInt(effectiveGasPrice)`, computed on the client; it is not stored. Note that prices are copied from the stored order, not from the chain event, which is safe because the poller first verifies the event's offer and consideration against the stored order (`verifyAgainstStored`) and refuses to record anything on a mismatch.

The indexes map directly onto intended queries: `(buyer, occurredAt)` for a buyer's history, `(seller, occurredAt)` for a seller's (no endpoint uses it yet), `(tokenContract, tokenId)` for per domain sale history (also unused so far), and `(occurredAt)` for the public recent feed. The unique `(orderHash, transactionType)` is, like history, the replay safety net.

```ts
// src/components/marketplacev2/transaction/enum/transaction-type.enum.ts
export enum TransactionType {
    SALE = 'SALE'
}
```

The comment is candid that the enum has a single value; the column exists so a future accepted offer, refund, or marketplace renewal can be added without a schema change. Uniqueness is on `(orderHash, transactionType)` rather than `orderHash` alone for the same reason: one order could one day have both a `SALE` and, say, a `REFUND`.

### How it is written, only ever by the poller

```ts
// src/components/marketplacev2/poller/application/seaport-event-application.service.ts
const gasInfo = await this.fetchGasInfo(log.transactionHash, orderHash);

const result = await this.orderRepository.manager.transaction(async (manager) => {
    const updateResult = await manager.update(
        OrderEntity,
        { orderHash, status: In(FILLABLE_FROM_STATUSES) },
        {
            status: OrderStatus.FILLED,
            fillTxHash: log.transactionHash,
            filledAt,
            filledBy: buyer
        }
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
    // ... then listing status UNLISTED and listing history SOLD, shown earlier
    return updateResult;
});
```

`FILLABLE_FROM_STATUSES` is `[ACTIVE, CANCELLED, INVALID, EXPIRED]`, because "the chain wins": an off chain cancel does not revoke the Seaport signature, so a soft cancelled order can still be filled on chain, and when it is, it must be recorded as a sale. In one transaction the order flips to `filled`, the transaction row is inserted, the status row flips to UNLISTED, and a SOLD history row is appended. If the order update affects zero rows (already filled by an earlier replay), nothing else is written. The gas lookup runs before and outside the transaction on purpose: it is network I/O that should not hold a database connection open, and its failure is non fatal.

```ts
// src/components/marketplacev2/poller/application/seaport-event-application.service.ts
private async fetchGasInfo(txHash: string, orderHash: string): Promise<{ gasUsed: string | null; effectiveGasPrice: string | null }> {
    try {
        const receipt = await retryWithBackoff(() => this.provider.getTransactionReceipt(txHash), { isRetryable: isRetryableChainError });
        if (!receipt) {
            this.customLoggerService.warn(`fetchGasInfo for ${orderHash} (tx ${txHash}) - receipt not found, recording the sale without gas info`);
            return { gasUsed: null, effectiveGasPrice: null };
        }
        return { gasUsed: receipt.gasUsed.toString(), effectiveGasPrice: receipt.gasPrice.toString() };
    } catch (err) {
        this.customLoggerService.warn(`fetchGasInfo for ${orderHash} (tx ${txHash}) failed - recording the sale without gas info: ${(err as Error).message}`);
        return { gasUsed: null, effectiveGasPrice: null };
    }
}
```

In ethers v6, `TransactionReceipt.gasPrice` is the effective gas price actually paid (what v5 called `effectiveGasPrice`), which is why it is stored under that name. It is fetched only for logs that already matched one of our orders, never for the flood of unrelated Seaport traffic on Polygon, and it is retried with backoff for transient RPC errors before degrading to `null`.

`TransactionService.record` itself is the same `.orIgnore()` insert as listing history:

```ts
// src/components/marketplacev2/transaction/transaction.service.ts
async record(manager: EntityManager, event: RecordTransactionEvent): Promise<void> {
    await manager.createQueryBuilder().insert().into(TransactionEntity).values({ ...event, createdAt: new Date() }).orIgnore().execute();
}
```

`TransactionModule` (`transaction/transaction.module.ts`) registers just the entity and `TransactionService`, exports the service, and is imported by both `OrderModule` (for the read routes) and `PollerModule` (for the single write site), the same sibling pattern as `ListingStatusModule`.

### How it is read: three endpoints

| Method and path | Guards | Query or params | Response `result` |
|---|---|---|---|
| `GET /transactions/recent` | none, public, `Cache-Control: public, max-age=60` | `RecentTransactionsQueryDto`: `limit?` (`@Type(() => Number) @IsInt @Min(1) @Max(100)`, default 12), `offset?` (`@IsInt @Min(0)`, default 0) | `{ data: TransactionReceiptDto[], total }` |
| `GET /transactions/mine` | `AccessTokenGuard`, `RequireVerifiedWalletGuard` | `MyTransactionsQueryDto`: `page?`, `limit?` (`@IsInt @Min(1)`, defaults 1 and 10) | `{ data: TransactionReceiptDto[], paginationInfo }` |
| `GET /transactions/:orderHash` | none, public | `orderHash` matching `0x[0-9a-fA-F]{64}` | `TransactionReceiptDto`, or 404 |

```ts
// src/components/marketplacev2/transaction/transaction.service.ts
async findByOrderHash(orderHash: string): Promise<TransactionEntity | null> {
    return this.transactionRepository.findOne({ where: { orderHash } });
}

async findRecent(skip: number, limit: number): Promise<{ data: TransactionEntity[]; total: number }> {
    const [data, total] = await this.transactionRepository.findAndCount({
        where: { transactionType: TransactionType.SALE },
        order: { occurredAt: 'DESC' },
        skip,
        take: limit
    });
    return { data, total };
}

async findByBuyer(buyerAddress: string, skip: number, limit: number): Promise<{ data: TransactionEntity[]; total: number }> {
    const qb = this.transactionRepository
        .createQueryBuilder('t')
        .where('LOWER(t."buyer") = LOWER(:buyer)', { buyer: buyerAddress })
        .andWhere('t."transactionType" = :transactionType', { transactionType: TransactionType.SALE });

    const total = await qb.getCount();
    const data = await qb.orderBy('t."occurredAt"', 'DESC').limit(limit).offset(skip).getMany();
    return { data, total };
}
```

The recent feed is the public "latest sales" strip, newest block time first, served by `idx_marketplacev2_transactions_occurred_at`. Its DTO's comment explains choosing plain `limit`/`offset` over a cursor: it is a bounded "recent N" widget, the same shape v1's own recent sales endpoint used, not a deep infinite scroll. The controller computes defaults and returns `{ data, total }` directly, without `paginateResponse`.

`transactions/mine` is the buyer's own receipts, the receipt table mirror of `GET /my-purchases` from `06`, adding `txHash`, `blockNumber`, `gasUsed`, and `effectiveGasPrice`. It deliberately uses `page`/`limit` with `paginateResponse` like the other "my" routes, but builds the pagination in the controller:

```ts
// src/components/marketplacev2/order/order.controller.ts
async myTransactions(@Query() query: MyTransactionsQueryDto, @Req() req: Request): Promise<Response> {
    const page = query.page ?? 1;
    const limit = query.limit ?? 10;
    const skip = getPaginationSkipByPageAndLimit(page, limit);
    const { data, total } = await this.transactionService.findByBuyer(req.walletAddress, skip, limit);
    return new Response(
        'marketplacev2 my transactions',
        paginateResponse(
            data.map((row) => this.transactionService.toReceiptDto(row)),
            total,
            new PaginationRequestDto(limit, skip, page)
        )
    );
}
```

The single receipt route returns a 404 `No transaction recorded for order 0x...` when no sale exists for the order, which covers orders still active, cancelled, invalid, or unknown. It is public because every field in a receipt is already visible on chain.

### The receipt DTO

```ts
// src/components/marketplacev2/transaction/dto/transaction-receipt.dto.ts
export interface TransactionReceiptDto {
    transactionType: TransactionType;
    orderHash: string;
    txHash: string;
    blockNumber: string;
    occurredAt: Date;
    chainId: number;
    domainName: string;
    tld: string;
    tokenId: string;
    tokenContract: string;
    buyer: string;
    seller: string;
    priceUsdt: string;
    feeUsdt: string;
    sellerUsdt: string;
    gasUsed: string | null;
    effectiveGasPrice: string | null;
}
```

`TransactionService.toReceiptDto(row)` is the one serializer every route uses, copying every column except `id` and `createdAt`, so the three endpoints can never drift in shape. In a React receipt card, format `priceUsdt` by dividing the `BigInt` by `10n ** 6n` (or use a decimal library), link `txHash` to Polygonscan, and render gas only when both gas fields are not `null`.

## Spec files

`src/components/marketplacev2/order/watchlist.service.spec.ts` mocks the two repositories. Under `add()`: TC4.1 creates a row with the checksum normalised contract and calls `save`; TC4.2 makes `save` reject with a unique violation and expects the existing row from `findOneOrFail({ where: { userId, tokenContract, tokenId } })` to be returned; and a non unique error such as "connection lost" is rethrown. Under `remove()`: TC4.7 expects `delete({ id, userId })`; TC4.8 (another user's row) and TC4.9 (nonexistent id) both expect `NotFoundException`. Under `list()`: an empty watchlist returns `[]`; TC4.3 a watched token with no order returns `live: false, order: null`; TC4.4 a servable order returns `live: true` with the summary; TC4.6 a relisted token picks the newest order rather than the older cancelled one; and a cancelled order with no relist returns `live: false` with the cancelled order still attached.

`src/components/marketplacev2/order/listing-status-sync.service.spec.ts` stubs `refreshDomainDetailData`, `findById`, and `syncAfterRefresh`. TC5.7 checks refresh runs first with the user id, then `findById`, then `syncAfterRefresh('user-1', ['1', '2'])`; another test checks domains with `token_id: null` are filtered out; TC5.11 checks everything is scoped to the passed user id; and the last checks `correctedCount` propagates unchanged.

`src/components/marketplacev2/listing-status/listing-status.service.spec.ts` checks `upsertListed` and `upsertUnlisted` call `upsert` with the right status and the conflict paths `['userId', 'domainName', 'tokenId']`, both through `manager.getRepository` rather than the injected repository; that `findAllForUser` reads `{ where: { userId } }` through its own repository; and under `syncAfterRefresh`, TC5.7 flips a LISTED row whose token is no longer owned (expecting `correctedCount: 1` and the exact `update` call), a still owned row is left alone with no `update` call, and the read is always `{ userId, status: LISTED }`.

`src/components/marketplacev2/listing-status/listing-history.service.spec.ts` checks `record` inserts through the passed manager with `createdAt` stamped; that a "genuine unique violation" resolves; and that other errors are rethrown. Its `findForUser` block (TC6.8, TC6.9) checks scoping by `userId` with `order: { occurredAt: 'DESC' }`, narrowing by `domainName`, by `tokenId`, by both as an AND, and that `skip`/`take` pass through.

`src/components/marketplacev2/transaction/transaction.service.spec.ts` mirrors that for transactions: `record` inserts with `createdAt` through the manager, resolves for a "unique violation", rethrows other errors; `findByOrderHash` reads `{ where: { orderHash } }`; `findRecent` scopes to `SALE`, `occurredAt DESC`, with the given `skip`/`take`, and returns rows and total as given; `findByBuyer` uses `LOWER(t."buyer") = LOWER(:buyer)` with `SALE`, newest first; and `toReceiptDto` maps every documented field without leaking `id` or `createdAt`.

The OrderService side of history (`listingHistory()` TC6.8 and TC6.9 in `order.service.spec.ts`) and the SOLD rule (`order.service.my-domains.spec.ts`) are described in `06`.

One honest note on the two "unique violation is a real ON CONFLICT DO NOTHING at the database level" tests: they mock `execute` to resolve and assert the promise resolves, but neither asserts that `orIgnore` was actually called (it is in the mock chain but never in an `expect`). So they would still pass if someone removed `.orIgnore()`, which is precisely the regression they are named after. Adding `expect(orIgnore).toHaveBeenCalled()` would make them mean what they say. Proving the actual Postgres behaviour would need an integration test against a real database.

## Bugs, risks, and inconsistencies

**Deleting a watchlist entry with a non UUID id is a 500.** `DELETE /watchlist/:id` has no `ParseUUIDPipe` (`order.controller.ts:353`), so `DELETE /watchlist/abc` sends `'abc'` to a `uuid` column and Postgres raises `invalid input syntax for type uuid`, surfacing as a 500 instead of the 404 the service is designed to return. The spec's `'someone-elses-row'` id only passes because the repository is mocked.

**A badly checksummed contract address on watchlist add is a 500.** `getAddress` (`watchlist.service.ts:23`) throws on an invalid mixed case checksum, outside the `try`, and the DTO's regex does not check checksums.

**The watchlist is unbounded and unvalidated.** Any contract and any numeric token id is accepted (`watchlist-add.dto.ts:6`), `@IsNumberString` also accepts values like `1.5`, there is no per user cap or rate limit, and `GET /watchlist` returns everything unpaginated, while `list` loads every order ever made for every watched token (`watchlist.service.ts:64`).

**Sync can wrongly unlist a still active listing.** `syncAfterRefresh` trusts whatever the refresh just wrote (`listing-status.service.ts:51`). If an Alchemy or Moralis call returns an incomplete set (indexing lag, a provider partially failing inside the refresh), a domain the user still owns and has actively listed is flipped to UNLISTED, while its order stays `active` and browsable, so my domains shows UNLISTED with no order for a domain that is in fact for sale, until the next create, cancel, or poller event for it.

**Sync crashes for a user without an EVM wallet.** The refresh it calls dereferences `evmNetworkWallet.walletAddress` without checking `evmNetworkWallet` exists (`domain-detail.service.ts:153`), and its `walletAddresses.length == null` guard can never be true (`domain-detail.service.ts:149`), so a Solana only user gets a 500 from sync.

**One rate limit budget is shared across three routes and is per process.** The single `apiCallLimiter` instance is mounted on forgot password, refresh domain, and sync (`main.ts:83`, `main.ts:84`, `main.ts:85`), so they share one per IP counter, and the default memory store counts per PM2 worker or instance.

**Listing history's index does not match its query.** The index is `(userId, createdAt)` (`listing-history.entity.ts:15`) but `findForUser` orders by `occurredAt` (`listing-history.service.ts:93`), so Postgres must sort the user's rows rather than read them in index order. An index on `(userId, occurredAt)` would fit.

**Offset pages have no tie breaker on time.** `findForUser`, `findRecent`, and `findByBuyer` order only by `occurredAt` (`listing-history.service.ts:93`, `transaction.service.ts:63`, `transaction.service.ts:88`). Several sales in the same Polygon block share an identical `occurredAt`, and Postgres does not guarantee a stable order among ties, so a row can appear on two pages or on none. Adding `id` (or `orderHash`) as a secondary sort fixes it.

**EXPIRED's `occurredAt` is sweep time, not expiry time.** `expireIfStale` records `occurredAt: new Date()` (`order.service.ts:582`), although the listing actually expired at its `endTime`, contradicting the entity's own rule that `occurredAt` is the real world moment this happened. After scheduler downtime, a listing that expired days ago shows as expiring today.

**Block time silently falls back to processing time.** Both Seaport handlers use `new Date()` when `log.blockTimestamp` is not a number (`seaport-event-application.service.ts:112` for fills, and the equivalent line in the cancel handler), so if the event source ever omits block timestamps, sale and cancel times quietly become poller times in both history and transactions.

**Legacy orders can lose history silently.** When an order has no `userId` and the case sensitive wallet lookup fails, status and history writes are skipped with only a log line (`seaport-event-application.service.ts:162` and the equivalent blocks in the other handlers).

**Buyer lookups cannot use their index.** `findByBuyer` filters `LOWER(t."buyer")` (`transaction.service.ts:84`), which cannot use `idx_marketplacev2_transactions_buyer_occurred_at`; an expression index on `LOWER(buyer)` or normalising `buyer` with `getAddress` at write time would. The entity comment says `buyer` is "Checksummed" (`transaction.entity.ts:66`) while the service comment says it is written raw with no `getAddress` pass, so the two comments contradict each other.

**The recent feed counts the whole table on every call.** `findAndCount` (`transaction.service.ts:61`) runs `COUNT(*)` over all sales per request, mitigated only by the 60 second cache header; `offset` has no maximum.

**Seller side and per domain sale history are indexed but not exposed.** `idx_marketplacev2_transactions_seller_occurred_at` and `idx_marketplacev2_transactions_token_contract_token_id` have no reading endpoint yet, so a seller cannot see their own receipts with gas data.

**History requires a fresh wallet signature without needing one.** `GET /listing-history` applies `RequireVerifiedWalletGuard` (`order.controller.ts:300`) although it queries only by `userId`, forcing a wallet re verification every hour to read one's own timeline, while the similar my domains and watchlist reads need only a JWT.

**Listing history filter is case sensitive.** `domainName` is matched with plain equality (`listing-history.service.ts:88`), so `?domainName=Fox.eth` returns nothing for a row stored as `fox.eth`.

## Frontend note

The pattern worth taking away from these four tables is the division of labour. The orders table is the truth; the listing status table is a cache shaped for one screen; the history table is an immutable diary; and the transaction table is an immutable ledger that only the chain itself (through the poller) is allowed to write. As a frontend developer you can lean on that: render "listed or not" from my domains, render "what happened" from listing history, render "what was paid" from transactions, and when in doubt about whether something is for sale right now, ask the order detail route, which is the only one that checks the real predicate at the moment you ask.
