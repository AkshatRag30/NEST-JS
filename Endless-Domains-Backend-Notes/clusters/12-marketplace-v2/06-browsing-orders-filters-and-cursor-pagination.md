# 06. Browsing Orders: Filters, Categories, and Cursor Pagination

## What this file covers, and where it sits in the v2 story

The other files in this cluster explain how a seller signs a Seaport order and how the backend validates and stores it in `tbl_marketplacev2_orders`, and how the poller later watches the chain and flips those rows to `filled`, `cancelled`, `invalid`, or `expired`. This file is about the other half of the life of that table, the read side. Once rows exist, who gets to see them, filtered how, sorted how, and paged how? There are seven read endpoints that matter here, all on `OrderController` under `/api/v1/marketplacev2/orders`, and between them they use three completely different pagination styles, which is itself one of the most instructive things in this whole module. The public browse page uses a genuine cursor based scheme, the "my something" pages use the same `page`/`limit` plus `paginateResponse` offset scheme the rest of the codebase has always used, and the public recent sales feed uses a bare `limit`/`offset` with a raw `{ data, total }` shape. By the end of this file you should be able to explain why each one was chosen, what each one costs, and exactly how a React page should drive each of them.

Most of this code landed in commit `64451cb4` ("Implemented new order api with different type of filter and cursor base pagination"), with the listing history read arriving in `225b5c86`, the transaction record table in `de4d906e`, and the transaction history routes in `008318b0`. Watchlist, listing history, and the transaction table themselves get their own deep dive in `07`; here they only appear where they touch browsing.

One stack note before diving in, because the project `CLAUDE.md` is now out of date on it: the `uat` branch is on NestJS 10 (`@nestjs/core ^10.4.22`), TypeORM `^0.3.17`, and ethers `^6.13.4`. That is why you will see TypeORM 0.3 idioms here (`FindOptionsWhere`, `findOneOrFail({ where })`, `.orIgnore()`, `repository.upsert`) and ethers v6 idioms (`getAddress` imported directly from `ethers`, `BigInt` everywhere instead of `BigNumber`).

## The endpoint map

Before reading any code, here is every read route this file discusses, with its guards and its pagination style. The full controller lives at `src/components/marketplacev2/order/order.controller.ts`.

| Method and path (under `/api/v1/marketplacev2/orders`) | Guards | Pagination | Service method |
|---|---|---|---|
| `GET /` (browse) | none, public, `Cache-Control: public, max-age=60` | cursor (`cursor`, `limit`), returns `nextCursor` | `OrderService.browse` |
| `GET /stats` | none, public, same cache header | none | `OrderService.stats` |
| `GET /my-domains` | `AccessTokenGuard` | offset (`page`, `limit`) via `paginateResponse` | `OrderService.myDomains` |
| `GET /my-listings` | `AccessTokenGuard`, `RequireVerifiedWalletGuard` | offset via `paginateResponse` | `OrderService.myListings` |
| `GET /my-purchases` | `AccessTokenGuard`, `RequireVerifiedWalletGuard` | offset via `paginateResponse` | `OrderService.myPurchases` |
| `GET /listing-history` | `AccessTokenGuard`, `RequireVerifiedWalletGuard` | offset via `paginateResponse` | `OrderService.listingHistory` (detail in `07`) |
| `GET /transactions/recent` | none, public, same cache header | raw `limit`/`offset`, returns `{ data, total }` | `TransactionService.findRecent` (detail in `07`) |
| `GET /transactions/mine` | `AccessTokenGuard`, `RequireVerifiedWalletGuard` | offset via `paginateResponse`, built in the controller | `TransactionService.findByBuyer` (detail in `07`) |
| `GET /transactions/:orderHash` | none, public | none | `TransactionService.findByOrderHash` (detail in `07`) |
| `GET /:orderHash` | none, public | none | `OrderService.findByHash` |
| `GET /_health/active` | `AccessTokenGuard`, `AdminTokenGuard` | none, hard capped at 50 | `OrderService.activeListingsHealth` |

Two guards in that table need a sentence each. `AccessTokenGuard` is the ordinary JWT check every logged in route uses. `RequireVerifiedWalletGuard` (`src/@core/common/guards/require-verified-wallet.guard.ts`) is new in v2: it reads `walletAddress` and `walletVerifiedAt` off the decoded JWT user, requires the verification to be no older than `WALLET_VERIFICATION_WINDOW_MS` (60 minutes, deliberately set equal to the access token lifetime on product's request, as the comment in that file records), and on success copies the address onto `request.walletAddress`. Every route that queries by wallet reads `req.walletAddress`, never something from the query string, which is what makes "my listings" mean the caller's listings and not anyone's.

### Route ordering and the `ORDER_HASH_ROUTE_PATTERN` trick

There is one routing detail worth understanding properly because it bites every Express based API eventually. Express matches routes in registration order, and a bare `@Get(':orderHash')` matches any single path segment, so if it were registered above `@Get('my-domains')`, a request for `/my-domains` would be routed to the detail handler with `orderHash = 'my-domains'`. The controller defends against this twice. First, it registers every literal route above the parameter routes, and the comments say so on each one. Second, and more robustly, it constrains the parameter itself:

```ts
// src/components/marketplacev2/order/constants/order-route.constants.ts
export const ORDER_HASH_ROUTE_PATTERN = '0x[0-9a-fA-F]{64}';
```

```ts
// src/components/marketplacev2/order/order.controller.ts
@Get(`:orderHash(${ORDER_HASH_ROUTE_PATTERN})`)
async getOrder(@Param('orderHash') orderHash: string): Promise<Response> {
    const order = await this.orderService.findByHash(orderHash);
    return new Response('Order found', this.toPublicOrder(order));
}
```

Because the path parameter only matches `0x` followed by exactly sixty four hex characters, `my-domains` structurally cannot match it no matter where it is declared, and a malformed hash returns a 404 from the router instead of reaching the service at all. The same pattern is reused inside `decodeCursor` to validate the tie breaker field of a cursor, which we will get to shortly.

The detail route is also the one place a signature is returned. `toPublicOrder` passes the whole `OrderEntity` through only when `isOrderServable(order)` is true (status `active` and `endTime` still in the future); otherwise it returns `{ ...order, signature: '' }`, blanking rather than deleting the field so the response shape stays stable for the client. A cancelled order's signature would otherwise remain fetchable forever and, since off chain cancellation does not revoke anything on Seaport, still fillable by anyone who fetched it. Browse never returns a signature at all, as you will see in `OrderSummaryDto`.

## The one predicate that defines "live": `applyServableWhere`

Every public read that means "currently for sale" goes through a single helper, and the doc comment is unusually explicit that nobody should ever inline it:

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

The reason status alone is not enough is that an order's `endTime` passing does not, by itself, change any row. `ListingExpiryScheduler` and `OrderService.expireIfStale` do eventually flip stale rows from `active` to `expired`, but they run on a schedule, so there is always a window where a row still says `active` but its Seaport order can no longer be filled. Checking `endTime > now` at read time makes that window invisible to users. `endTime` is a Postgres `bigint` column holding Unix seconds, and `nowSeconds` is passed as a string parameter that Postgres coerces to `bigint` for the comparison, so this is a true numeric comparison, not a string one. The SQL form is used by browse, stats, and the live order lookup inside my domains; the in memory twin `isOrderServable` is used by the detail route's signature blanking and by the watchlist join. Keeping both in one file is the whole defence against the two ever disagreeing.

## Browse: `GET /api/v1/marketplacev2/orders`

### The query DTO, field by field

```ts
// src/components/marketplacev2/order/dto/order-browse-query.dto.ts
export const BROWSE_DEFAULT_PAGE_SIZE = 24;
export const BROWSE_MAX_PAGE_SIZE = 100;

export const ORDER_BROWSE_SORTS = ['recent', 'price_asc', 'price_desc'] as const;
export type OrderBrowseSort = (typeof ORDER_BROWSE_SORTS)[number];

export class OrderBrowseQueryDto {
    @IsOptional() @IsString() tld?: string;
    @IsOptional() @IsNumberString() maxPrice?: string;
    @IsOptional() @IsString() q?: string;
    @IsOptional() @IsIn(ORDER_BROWSE_SORTS) sort?: OrderBrowseSort;
    @IsOptional() @IsIn(DOMAIN_CATEGORY_FILTERS) category?: DomainCategory;
    @IsOptional() @IsString() cursor?: string;
    @IsOptional() @Type(() => Number) @IsInt() @Min(1) @Max(BROWSE_MAX_PAGE_SIZE) limit?: number;
}
```

(The real file puts each decorator on its own line; they are condensed here so the whole shape fits in one view.)

| Param | Validator | Default (applied in the service) | Meaning |
|---|---|---|---|
| `tld` | `@IsString` | none | Exact TLD match, compared case insensitively via `LOWER(o."tld") = LOWER(:tld)` |
| `maxPrice` | `@IsNumberString` | none | Upper bound on `priceUsdt`, in USDT minor units (6 decimals), so `$25` is `25000000` |
| `q` | `@IsString` | none | Case insensitive substring search on `domainName` via `ILIKE` |
| `sort` | `@IsIn(['recent','price_asc','price_desc'])` | `'recent'` | Ordering, also decides what the cursor's `sortKey` holds |
| `category` | `@IsIn(['all','short','numeric','dictionary','brandable','premium'])` | `'all'` | The tab the user is on |
| `cursor` | `@IsString` | none (first page) | Opaque token from a previous response's `nextCursor` |
| `limit` | `@Type(() => Number) @IsInt @Min(1) @Max(100)` | `24` | Page size; above 100 is a 400, never silently clamped |

The `@Type(() => Number)` on `limit` matters because query strings are always strings; combined with the global `ValidationPipe({ whitelist: true, transform: true })` in `main.ts` it converts `"24"` into the number `24` before `@IsInt` runs. `maxPrice`, by contrast, deliberately stays a string, since prices in this module are always minor unit decimal strings that may exceed what a JS number can hold exactly. One small gap: `@IsNumberString()` with no options accepts things like `"1.5"` and `"-3"`. A decimal would make `:maxPrice::bigint` throw a Postgres cast error (a 500 rather than a 400), and a negative value simply returns nothing, so neither is dangerous, but `@Matches(/^\d+$/)` would turn the first into a clean 400.

### The response shape: `PaginatedOrdersDto` and `OrderSummaryDto`

```ts
// src/components/marketplacev2/order/dto/paginated-orders.dto.ts
export interface CategoryCounts {
    all: number;
    short: number;
    numeric: number;
    dictionary: number;
    brandable: number;
    premium: number;
}

export interface PaginatedOrdersDto {
    items: OrderSummaryDto[];
    nextCursor: string | null;
    categoryCounts?: CategoryCounts;
}
```

```ts
// src/components/marketplacev2/order/dto/order-summary.dto.ts
export class OrderSummaryDto {
    orderHash: string;
    domainName: string;
    tld: string;
    tokenId: string;
    priceUsdt: string;
    feeUsdt: string;
    sellerUsdt: string;
    maker: string;
    status: OrderStatus;
    startTime: string;
    endTime: string;
    createdAt: Date;
    filledAt: Date | null;
    fillTxHash: string | null;
}
```

Wrapped in the standard envelope, a browse response looks like `{ message: 'marketplacev2 order browse', result: { items, nextCursor, categoryCounts } }`. Notice what is absent: there is no `total`, no `page`, no `lastPage`. That is not an oversight, it is the defining property of cursor pagination, which we will dig into in its own section. Also notice what `OrderSummaryDto` omits compared with `OrderEntity`: `signature`, `rawOrder`, `counter`, `salt`, `conduitKey`, `chainId`, `tokenContract`, `filledBy`, `invalidReason`, `userId`. The single function `toOrderSummary(row)` is "the one place OrderEntity is narrowed", shared by browse and the watchlist, so the public shape cannot drift between the two. To actually buy, a frontend takes `orderHash` from a browse item and calls `GET /:orderHash`, which returns the signature only if the order is still servable at that moment.

The field meanings: `priceUsdt` is the total the buyer pays, `feeUsdt` the marketplace fee, and `sellerUsdt` what the seller receives, all USDT minor units as decimal strings with `sellerUsdt + feeUsdt == priceUsdt` enforced at creation. `startTime` and `endTime` are Unix seconds as strings (they come from `bigint` columns through a pass through transformer), `createdAt` is the server time the order was accepted, and `filledAt`/`fillTxHash` are always `null` on browse because browse only ever returns active rows.

### The top level `browse()` method

```ts
// src/components/marketplacev2/order/order.service.ts
async browse(query: OrderBrowseQuery): Promise<PaginatedOrdersDto> {
    const limit = query.limit ?? BROWSE_DEFAULT_PAGE_SIZE;
    const sort: OrderBrowseSort = query.sort ?? 'recent';
    const category: DomainCategory = query.category ?? 'all';
    const cursor = query.cursor ? decodeCursor(query.cursor) : null;

    let items: OrderSummaryDto[];
    let nextCursor: string | null;
    let scanRows: OrderEntity[] | null = null;

    if (isSqlDomainCategory(category)) {
        ({ items, nextCursor } = await this.browseSqlNative(query, category, sort, limit, cursor));
    } else {
        scanRows = await this.scanServableRows(query, sort);
        ({ items, nextCursor } = this.browseInApp(scanRows, category, sort, limit, cursor));
    }

    const categoryCounts = cursor ? undefined : await this.computeCategoryCounts(query, scanRows);

    return { items, nextCursor, categoryCounts };
}
```

Read it as a fork. The cursor is decoded and validated first, so a bad cursor fails before any query runs. Then the method asks one question about the category: can it be expressed as a plain SQL `WHERE`? For `all`, `short`, `numeric`, and `premium` the answer is yes, and the whole page is computed by Postgres. For `dictionary` and `brandable` the answer is no (you cannot ask Postgres "is this label an English word" without a word table), so the method fetches a large, capped, presorted scan of servable rows and filters and pages it in Node. Finally, tab counts are computed only when there is no cursor, meaning only on the first page of a given filter set. That is a nice piece of judgement: the tabs only need fresh numbers when the user lands on the page or changes a filter, and every subsequent infinite scroll fetch is kept as cheap as possible.

### The shared base query: `buildBrowseBaseQb`

```ts
// src/components/marketplacev2/order/order.service.ts
private buildBrowseBaseQb(query: OrderBrowseQuery): SelectQueryBuilder<OrderEntity> {
    const qb = this.orderRepository.createQueryBuilder('o');
    applyServableWhere(qb, 'o');

    if (query.tld) {
        qb.andWhere('LOWER(o."tld") = LOWER(:tld)', { tld: query.tld });
    }
    if (query.maxPrice !== undefined) {
        qb.andWhere('o."priceUsdt"::bigint <= :maxPrice::bigint', { maxPrice: query.maxPrice });
    }
    if (query.q) {
        const escapedSearch = query.q.replace(/[\\%_]/g, (ch) => `\\${ch}`);
        qb.andWhere(`o."domainName" ILIKE :q ESCAPE '\\'`, { q: `%${escapedSearch}%` });
    }
    return qb;
}
```

Every input reaches SQL as a bound parameter, so there is no SQL injection path here. The `q` handling goes one step further than most code in this repository and is worth copying in your own work: inside a `LIKE`/`ILIKE` pattern, `%` means "any run of characters" and `_` means "any single character", so a user searching for `a_b` without escaping would also match `axb`. By prefixing `\`, `%`, and `_` with a backslash and declaring `ESCAPE '\'`, the user's text is treated literally and only the wrapping `%...%` acts as a wildcard. This is not a security issue (parameters already prevent injection), it is a correctness issue, and it is the kind of detail that separates a search box that "mostly works" from one that is exact.

The SQL this produces for, say, `?tld=eth&maxPrice=50000000&q=fox` is approximately:

```ts
// approximate SQL generated by buildBrowseBaseQb (illustrative)
SELECT o.* FROM tbl_marketplacev2_orders o
WHERE o.status = 'active'
  AND o.endTime > '1790000000'
  AND LOWER(o."tld") = LOWER('eth')
  AND o."priceUsdt"::bigint <= '50000000'::bigint
  AND o."domainName" ILIKE '%fox%' ESCAPE '\'
```

On indexes: the entity declares `idx_marketplacev2_orders_status`, `idx_marketplacev2_orders_maker`, `idx_marketplacev2_orders_token_id`, the composite `idx_marketplacev2_orders_status_created_at` on `(status, createdAt)`, and the partial unique `idx_marketplacev2_orders_active_listing_unique` on `(tokenContract, tokenId) WHERE status = 'active'`. The `(status, createdAt)` composite is the one that genuinely helps browse, because the default sort is `createdAt DESC` within `status = 'active'`. Nothing indexes `LOWER(tld)`, the `ILIKE '%...%'` search (a leading wildcard can never use a plain btree), or `priceUsdt::bigint` ordering, so price sorts and searches are sequential scans plus a sort over the active set. At today's volume that is fine; a `pg_trgm` GIN index on `domainName` and an index on `(status, priceUsdt, orderHash)` would be the natural next steps once the active set grows into the tens of thousands.

### Sorting with a tuple cursor: `applyBrowseSort`

```ts
// src/components/marketplacev2/order/order.service.ts
private applyBrowseSort(qb: SelectQueryBuilder<OrderEntity>, sort: OrderBrowseSort, cursor: OrderCursor | null): void {
    switch (sort) {
        case 'price_asc':
            if (cursor) {
                qb.andWhere('(o."priceUsdt"::bigint, o."orderHash") > (:cursorPrice::bigint, :cursorOrderHash)', { cursorPrice: cursor.sortKey, cursorOrderHash: cursor.orderHash });
            }
            qb.orderBy('o."priceUsdt"::bigint', 'ASC').addOrderBy('o."orderHash"', 'ASC');
            return;
        case 'price_desc':
            if (cursor) {
                qb.andWhere('(o."priceUsdt"::bigint, o."orderHash") < (:cursorPrice::bigint, :cursorOrderHash)', { cursorPrice: cursor.sortKey, cursorOrderHash: cursor.orderHash });
            }
            qb.orderBy('o."priceUsdt"::bigint', 'DESC').addOrderBy('o."orderHash"', 'DESC');
            return;
        case 'recent':
        default:
            if (cursor) {
                qb.andWhere('(o."createdAt", o."orderHash") < (:cursorCreatedAt::timestamp, :cursorOrderHash)', { cursorCreatedAt: cursor.sortKey, cursorOrderHash: cursor.orderHash });
            }
            qb.orderBy('o."createdAt"', 'DESC').addOrderBy('o."orderHash"', 'DESC');
    }
}
```

This is the heart of the cursor design and it is worth slowing down for. Each sort orders by its primary key (`priceUsdt` or `createdAt`) and then by `orderHash` as a tie breaker. The tie breaker is what makes the ordering total: two listings can easily share a price (lots of domains are listed at a round `$10`), but no two orders share an `orderHash`, since it is the primary key. A total order is a precondition for a correct cursor, because a cursor says "give me everything strictly after this exact position", and "this exact position" is ambiguous if two rows compare equal.

Then the comparison itself. `(a, b) > (x, y)` is a Postgres row value comparison and means "a > x, or a = x and b > y", which is exactly lexicographic order. The comment in `order-cursor.util.ts` explains why this must not be written as two separate conditions, and it is a classic bug worth internalising. The naive version, `priceUsdt >= :p AND orderHash > :h`, looks right but silently drops rows: suppose the last row of page one is `($10, 0x9f...)` and the next row is `($12, 0x01...)`. The naive filter keeps only rows whose hash is greater than `0x9f...`, so `($12, 0x01...)` is skipped forever even though its price is higher. The tuple form compares the price first and only consults the hash when prices tie, which is exactly the ordering the `ORDER BY` uses. The rule to remember is that the cursor predicate must be the precise mirror of the `ORDER BY`, column for column, direction for direction, and here it is.

The `:cursorPrice::bigint` syntax is a TypeORM named parameter immediately followed by a Postgres cast. TypeORM 0.3's parameter parser uses a negative lookbehind for `:`, so `::bigint` is correctly left alone as a cast rather than being mistaken for a second parameter.

### The cursor itself: `order-cursor.util.ts`

```ts
// src/components/marketplacev2/order/util/order-cursor.util.ts
export interface OrderCursor {
    sortKey: string;
    orderHash: string;
}

const ORDER_HASH_PATTERN = new RegExp(`^${ORDER_HASH_ROUTE_PATTERN}$`);

export function encodeCursor(cursor: OrderCursor): string {
    return Buffer.from(JSON.stringify(cursor), 'utf8').toString('base64url');
}

export function decodeCursor(raw: string): OrderCursor {
    let parsed: unknown;
    try {
        parsed = JSON.parse(Buffer.from(raw, 'base64url').toString('utf8'));
    } catch {
        throw new BadRequestException('Invalid cursor');
    }

    if (typeof parsed !== 'object' || parsed === null) {
        throw new BadRequestException('Invalid cursor');
    }

    const { sortKey, orderHash } = parsed as Record<string, unknown>;
    if (typeof sortKey !== 'string' || typeof orderHash !== 'string' || !ORDER_HASH_PATTERN.test(orderHash)) {
        throw new BadRequestException('Invalid cursor');
    }

    return { sortKey, orderHash };
}
```

And the place a cursor is minted from the last row of a page:

```ts
// src/components/marketplacev2/order/order.service.ts
private cursorFor(row: OrderEntity, sort: OrderBrowseSort): OrderCursor {
    return { sortKey: sort === 'recent' ? row.createdAt.toISOString() : row.priceUsdt, orderHash: row.orderHash };
}
```

So a cursor is nothing more than JSON like `{"sortKey":"2026-09-30T12:01:02.345Z","orderHash":"0xab..."}` encoded with base64url (the URL safe alphabet that uses `-` and `_` instead of `+` and `/` and drops padding, so it can sit in a query string without percent encoding). It is opaque by convention, not by cryptography: anyone can decode it, and that is fine, because it reveals nothing that is not already in the response. What `decodeCursor` does guarantee is shape: it must be base64url, must decode to JSON, must be an object, must have a string `sortKey` and a string `orderHash`, and the hash must match the same `0x` plus sixty four hex pattern the routes use. Anything else is a clean 400 "Invalid cursor".

What it does not validate is whether `sortKey` makes sense for the requested sort, and that is a real gap, covered in the bugs section below.

### Path one: SQL native categories (`browseSqlNative`)

```ts
// src/components/marketplacev2/order/order.service.ts
private async browseSqlNative(query: OrderBrowseQuery, category: SqlDomainCategory, sort: OrderBrowseSort, limit: number, cursor: OrderCursor | null): Promise<{ items: OrderSummaryDto[]; nextCursor: string | null }> {
    const qb = this.buildBrowseBaseQb(query);
    const categoryWhere = buildSqlCategoryWhere(category, 'o');
    if (categoryWhere) {
        qb.andWhere(categoryWhere);
    }
    this.applyBrowseSort(qb, sort, cursor);
    qb.limit(limit + 1);

    const rows = await qb.getMany();
    const hasMore = rows.length > limit;
    const pageRows = hasMore ? rows.slice(0, limit) : rows;

    return {
        items: pageRows.map((row) => this.toOrderSummary(row)),
        nextCursor: hasMore ? encodeCursor(this.cursorFor(pageRows[pageRows.length - 1], sort)) : null
    };
}
```

The "fetch `limit + 1`" trick is the standard way to know whether a next page exists without running a second `COUNT(*)`. Ask for 25 when the page is 24; if 25 come back, there is more, so drop the extra one and mint a cursor from row 24; if 24 or fewer come back, this is the last page and `nextCursor` is `null`. It costs one extra row over the wire and saves an entire query, and it is precisely why the response has no `total`.

### Path two: in application categories (`scanServableRows` and `browseInApp`)

```ts
// src/components/marketplacev2/order/order.service.ts
private static readonly DICTIONARY_SCAN_CAP = 5000;

private async scanServableRows(query: OrderBrowseQuery, sort: OrderBrowseSort): Promise<OrderEntity[]> {
    const qb = this.buildBrowseBaseQb(query);
    this.applyBrowseSort(qb, sort, null);
    qb.limit(OrderService.DICTIONARY_SCAN_CAP);

    const rows = await qb.getMany();
    if (rows.length === OrderService.DICTIONARY_SCAN_CAP) {
        this.customLoggerService.warn(`marketplacev2 browse: dictionary/brandable scan hit its ${OrderService.DICTIONARY_SCAN_CAP}-row cap - classification may be incomplete at the current table size`);
    }
    return rows;
}

private browseInApp(scanRows: OrderEntity[], category: AppDomainCategory, sort: OrderBrowseSort, limit: number, cursor: OrderCursor | null): { items: OrderSummaryDto[]; nextCursor: string | null } {
    const matching = scanRows.filter((row) => {
        const flags = classifyDomainCategories(row.domainName, row.tld);
        return category === 'dictionary' ? flags.dictionary : flags.brandable;
    });

    const startIndex = cursor ? this.findCursorIndex(matching, sort, cursor) : 0;
    const pageRows = matching.slice(startIndex, startIndex + limit);
    const hasMore = startIndex + limit < matching.length;

    return {
        items: pageRows.map((row) => this.toOrderSummary(row)),
        nextCursor: hasMore ? encodeCursor(this.cursorFor(pageRows[pageRows.length - 1], sort)) : null
    };
}
```

Here the database does the cheap filtering (servable, tld, maxPrice, q) and the sorting, but not the category or the cursor. Up to 5,000 rows come back, Node classifies each one, keeps the dictionary (or brandable) matches, and then finds where the cursor falls inside that already sorted list:

```ts
// src/components/marketplacev2/order/order.service.ts
private findCursorIndex(rows: OrderEntity[], sort: OrderBrowseSort, cursor: OrderCursor): number {
    const index = rows.findIndex((row) => this.isAfterCursor(row, sort, cursor));
    return index === -1 ? rows.length : index;
}

private isAfterCursor(row: OrderEntity, sort: OrderBrowseSort, cursor: OrderCursor): boolean {
    switch (sort) {
        case 'price_asc': {
            const rowPrice = BigInt(row.priceUsdt);
            const cursorPrice = BigInt(cursor.sortKey);
            return rowPrice !== cursorPrice ? rowPrice > cursorPrice : row.orderHash > cursor.orderHash;
        }
        case 'price_desc': {
            const rowPrice = BigInt(row.priceUsdt);
            const cursorPrice = BigInt(cursor.sortKey);
            return rowPrice !== cursorPrice ? rowPrice < cursorPrice : row.orderHash < cursor.orderHash;
        }
        case 'recent':
        default: {
            const rowKey = row.createdAt.toISOString();
            return rowKey !== cursor.sortKey ? rowKey < cursor.sortKey : row.orderHash < cursor.orderHash;
        }
    }
}
```

`isAfterCursor` is a hand written JavaScript copy of the SQL tuple comparison, so in app pages line up exactly with what the SQL path would have produced. Prices are compared as `BigInt` (never as JS numbers, which lose precision above 2^53), and `recent` compares ISO strings, which works because ISO 8601 UTC strings of fixed width sort lexicographically in chronological order. Using `findIndex` rather than "find the row whose hash equals the cursor's hash" is also a thoughtful choice: if the cursor's own row has since been sold or cancelled and dropped out of the scan, `findIndex` still lands on the first row after where it used to be, so the scroll continues instead of restarting.

The honest cost is that every single dictionary or brandable page, including page five of an infinite scroll, runs the 5,000 row scan again and reclassifies all of it. It is a deliberate tradeoff the comments own up to, with a warning log as a tripwire, but the tripwire only fires when the cap is hit, and by then results are already silently truncated: listings beyond row 5,000 of the scan simply never appear under these two tabs, and the infinite scroll ends early with `nextCursor: null` as though there were nothing more.

### Tab counts: `computeCategoryCounts`

```ts
// src/components/marketplacev2/order/order.service.ts
private async computeCategoryCounts(query: OrderBrowseQuery, scanRows: OrderEntity[] | null): Promise<CategoryCounts> {
    const sqlCategories: SqlDomainCategory[] = ['all', 'short', 'numeric', 'premium'];
    const [all, short, numeric, premium] = await Promise.all(
        sqlCategories.map((category) => {
            const qb = this.buildBrowseBaseQb(query);
            const where = buildSqlCategoryWhere(category, 'o');
            if (where) {
                qb.andWhere(where);
            }
            return qb.getCount();
        })
    );

    const rows = scanRows ?? (await this.scanServableRows(query, 'recent'));
    let dictionary = 0;
    let brandable = 0;
    for (const row of rows) {
        const flags = classifyDomainCategories(row.domainName, row.tld);
        if (flags.dictionary) dictionary++;
        if (flags.brandable) brandable++;
    }

    return { all, short, numeric, dictionary, brandable, premium };
}
```

The counts respect `tld`, `maxPrice`, and `q` (they all start from `buildBrowseBaseQb(query)`), so the numbers on each tab always describe "how many would I see if I clicked this tab with my current filters", which is the UX a user expects. They ignore `category` itself, which is also right, since each tab counts its own category. Do note the query budget of a first page, though: on the `all` tab it is one page query, four parallel `COUNT(*)` queries, and one 5,000 row scan with classification, six queries in all. Exact SQL counts sitting next to scan derived dictionary and brandable counts also means that once the active set passes 5,000, the tabs quietly become inconsistent, for example `all: 8000` but `dictionary + brandable + short + numeric` adding up to far less than it should.

## Category classification in depth: `domain-category.util.ts`

None of these categories is a column. They are all derived at read time from `domainName` and `tld`. The file's doc comment lays out the split cleanly: short, numeric, and premium are cheap SQL predicates; dictionary has no sane single query SQL form against a roughly 275,000 word list without a persisted column, so it, and its complement brandable, are classified in Node.

```ts
// src/components/marketplacev2/order/util/domain-category.util.ts
export const DOMAIN_CATEGORY_FILTERS = ['all', 'short', 'numeric', 'dictionary', 'brandable', 'premium'] as const;
export type SqlDomainCategory = Extract<DomainCategory, 'all' | 'short' | 'numeric' | 'premium'>;
export type AppDomainCategory = Extract<DomainCategory, 'dictionary' | 'brandable'>;

export const PREMIUM_TLDS: string[] = [];

export function extractLabel(domainName: string, tld: string): string {
    const suffix = `.${tld}`;
    return domainName.toLowerCase().endsWith(suffix.toLowerCase()) ? domainName.slice(0, domainName.length - suffix.length) : domainName;
}

export function extractLabelSql(alias: string): string {
    return `LEFT(${alias}."domainName", LENGTH(${alias}."domainName") - LENGTH(${alias}."tld") - 1)`;
}

export function isShortLabel(label: string): boolean {
    return label.length === 2 || label.length === 3;
}

export function isNumericLabel(label: string): boolean {
    return /[0-9]/.test(label);
}

export function isPremiumTld(tld: string): boolean {
    return PREMIUM_TLDS.some((premiumTld) => premiumTld.toLowerCase() === tld.toLowerCase());
}
```

The rules, stated plainly:

| Category | Rule | Where evaluated | Notes |
|---|---|---|---|
| `all` | no filter | SQL (returns `null` predicate) | |
| `short` | label length is exactly 2 or 3 | SQL `LENGTH(label) IN (2, 3)` and JS | single character labels are not short |
| `numeric` | label contains at least one digit anywhere | SQL `label ~ '[0-9]'` and JS `/[0-9]/` | so `crypto1` is numeric, not just `123` |
| `premium` | TLD is in `PREMIUM_TLDS` | SQL `LOWER(tld) = ANY(ARRAY[...]::text[])` | list is empty today, so always zero |
| `dictionary` | label is exactly an English word, case insensitive | JS only | exact match, `flowerpot` is not `flower` |
| `brandable` | none of short, numeric, dictionary, premium | JS only | the catch all |

The "label" is the part before `.tld`. The JS version uses the separate `tld` column to strip the suffix (so a name like `my.sub.eth` with `tld = 'eth'` yields `my.sub`, not `my`), falling back to the whole name if it does not end with `.tld`. The SQL version is purely arithmetic, chopping `LENGTH(tld) + 1` characters off the end without checking that the suffix actually matches. For well formed rows the two agree; for a row whose `domainName` does not end with its own `tld` they would diverge, a small edge the spec does not cover.

Categories are not mutually exclusive. `go.com` is both short and dictionary; `a1.com` is both short and numeric. Only brandable is defined as exclusive. So the tab counts should not be expected to sum to `all`, and a frontend should not present them as a breakdown.

The premium case is worth a second look because of how the empty list is handled:

```ts
// src/components/marketplacev2/order/util/domain-category.util.ts
case 'premium': {
    const list = PREMIUM_TLDS.map((tld) => `'${tld.toLowerCase().replace(/'/g, "''")}'`).join(', ');
    return `LOWER(${alias}."tld") = ANY(ARRAY[${list}]::text[])`;
}
```

This is the only place in the browse path where values are interpolated into SQL text instead of bound as parameters. It is safe today because `PREMIUM_TLDS` is a hardcoded constant, not user input, and single quotes are doubled anyway. With an empty list it produces `ANY(ARRAY[]::text[])`, which is always false, so the premium tab shows zero rather than erroring, which is exactly what the spec asserts. If anyone ever sources `PREMIUM_TLDS` from an admin setting or a database row, this should be switched to a bound array parameter.

### The dictionary and the `an-array-of-english-words` dependency

```ts
// src/components/marketplacev2/order/util/domain-category.util.ts
let dictionaryWords: Set<string> | null = null;

function getDictionaryWords(): Set<string> {
    if (!dictionaryWords) {
        // eslint-disable-next-line @typescript-eslint/no-var-requires
        const words: string[] = require('an-array-of-english-words');
        dictionaryWords = new Set(words.map((word) => word.toLowerCase()));
    }
    return dictionaryWords;
}

export function isDictionaryLabel(label: string): boolean {
    return getDictionaryWords().has(label.toLowerCase());
}

export function classifyDomainCategories(domainName: string, tld: string): DomainCategoryFlags {
    const label = extractLabel(domainName, tld);
    const short = isShortLabel(label);
    const numeric = isNumericLabel(label);
    const premium = isPremiumTld(tld);
    const dictionary = isDictionaryLabel(label);
    const brandable = !short && !numeric && !dictionary && !premium;
    return { short, numeric, dictionary, premium, brandable };
}
```

`an-array-of-english-words` (`^2.0.0`, resolved to `2.0.0` in `package-lock.json`, added to `package.json` by commit `64451cb4`) is a package that is literally one big array of roughly 275,000 lowercase English words. The service loads it lazily on first use with a synchronous `require`, turns it into a `Set` (so each lookup is constant time instead of scanning an array), and caches it in a module level variable for the life of the process. Two practical consequences: the very first dictionary or brandable request after a deploy pays the parse and `Set` build cost inline on the request path (tens of milliseconds and a noticeable chunk of heap), and every PM2 worker pays it once separately. The list is also very permissive, it includes a lot of obscure words and many two letter entries, so plenty of labels a human would call brandable will land in dictionary. That is a product question, not a bug, but it is worth knowing before someone files "why is `qi.eth` a dictionary word" as one. Since `node_modules` was not available to inspect, one thing worth confirming once locally is that this major version still exposes a plain array through CommonJS `require`, since the spec's `isDictionaryLabel('flower') === true` assertion is what would catch it if not.

## The stats strip: `GET /api/v1/marketplacev2/orders/stats`

```ts
// src/components/marketplacev2/order/order.service.ts
async stats(): Promise<OrderStatsDto> {
    const liveCountQb = this.orderRepository.createQueryBuilder('o');
    applyServableWhere(liveCountQb, 'o');

    const medianQb = this.orderRepository.createQueryBuilder('o').select('PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY o."priceUsdt"::bigint)::text', 'medianAsk');
    applyServableWhere(medianQb, 'o');

    const filledQb = this.orderRepository
        .createQueryBuilder('o')
        .select('COALESCE(SUM(o."priceUsdt"::bigint), 0)::text', 'totalVolume')
        .addSelect('COUNT(*)', 'salesCount')
        .where('o.status = :filledStatus', { filledStatus: OrderStatus.FILLED });

    const [liveCount, medianRow, filledRow, cursorRow] = await Promise.all([
        liveCountQb.getCount(),
        medianQb.getRawOne<{ medianAsk: string | null }>(),
        filledQb.getRawOne<{ totalVolume: string; salesCount: string }>(),
        this.pollerCursorRepository.findOne({ where: { id: SINGLETON_CURSOR_ID } })
    ]);

    return {
        liveCount,
        totalVolume: filledRow?.totalVolume ?? '0',
        salesCount: Number(filledRow?.salesCount ?? 0),
        medianAsk: medianRow?.medianAsk ?? null,
        lastIndexedBlock: cursorRow ? Number(cursorRow.lastProcessedBlock) : 0
    };
}
```

```ts
// src/components/marketplacev2/order/dto/order-stats.dto.ts
export interface OrderStatsDto {
    liveCount: number;
    totalVolume: string;
    salesCount: number;
    medianAsk: string | null;
    lastIndexedBlock: number;
}
```

Four independent queries run in parallel. `liveCount` and `medianAsk` use the servable predicate, so they match browse exactly; `totalVolume` and `salesCount` count every `filled` order ever, regardless of `endTime`, because a sale is a permanent fact. `medianAsk` is `null`, never `"0"`, when there are no live listings, so a UI can show a dash instead of a misleading `$0`. `lastIndexedBlock` comes from the poller's singleton cursor row (`tbl_marketplacev2_poller_cursor`, see the poller file in this cluster) and lets a UI show "indexed up to block N", or compare against the current chain head to show "data is X blocks behind". `COUNT(*)` comes back from the node postgres driver as a string, hence the `Number(...)`. A subtlety: `totalVolume` here is computed from the orders table, while the transaction table (`07`) also holds every sale price; the two should agree, but they are two separate sources of truth for the same number.

One real wrinkle: `PERCENTILE_CONT` interpolates and returns `double precision`. With an even number of live listings the median is the average of the middle two, so prices `100` and `201` produce `"150.5"`. Every other amount in the module is an integer minor unit string, and a frontend that does `BigInt(stats.medianAsk)` (the natural thing to do with this module's conventions) will throw `SyntaxError: Cannot convert 150.5 to a BigInt`. `PERCENTILE_DISC(0.5)` (which returns an actual element) or wrapping the result in `ROUND(...)::bigint::text` would keep the convention intact.

## The "my" endpoints and offset pagination

### How `paginateResponse` works, and what the response looks like

All the "my" routes return the same shape the rest of the codebase has used for years, built by the helper in `src/@core/utils/helper.ts`:

```ts
// src/@core/utils/helper.ts
export function paginateResponse(data: any, total: number, paginationRequest: PaginationRequestDto): ResultResponse {
    const limit = paginationRequest?.limit || 10;
    const page = paginationRequest?.page || 1;
    const lastPage = Math.ceil(total / limit);
    const nextPage = page + 1 > lastPage ? null : page + 1;
    const prevPage = page - 1 < 1 ? null : page - 1;
    return new ResultResponse([...data]).setPaginationInfo(new PaginationInfo(total, page, nextPage, prevPage, lastPage));
}

export function getPaginationSkipByPageAndLimit(page: number, limit: number): number {
    return (page - 1) * limit;
}
```

So the envelope is `{ message, result: { data: [...], paginationInfo: { total, page, nextPage, prevPage, lastPage } } }`. Note the property is `data` (the `ResultResponse` class), even though the project `CLAUDE.md` calls it `result`. Every one of the four query DTOs below repeats the same `page`/`limit` validators: `@IsOptional() @Type(() => Number) @IsInt() @Min(1)`, defaulting to `page = 1`, `limit = 10` in the service. None of them has a `@Max`, unlike browse's cap of 100, which is the first inconsistency worth noting: `?limit=100000` is accepted on every "my" route.

### `GET /my-domains`: current ownership, annotated with listing status

This endpoint answers "what do I own right now, and which of those are listed, unlisted, or sold?". Its DTO:

```ts
// src/components/marketplacev2/order/dto/my-domains-query.dto.ts
const DOMAIN_EXPIRY_STATUS_VALUES: DomainExpiryStatus[] = ['Expired', 'GracePeriod', 'ExpiringSoon', 'Normal'];

export class MyDomainsQueryDto {
    @IsOptional() @IsIn(MY_DOMAINS_STATUS_FILTER_VALUES) status?: MyDomainsStatusFilter;
    @IsOptional() @IsIn(DOMAIN_EXPIRY_STATUS_VALUES) expiryStatus?: DomainExpiryStatus;
    @IsOptional() @Type(() => Number) @IsInt() @Min(1) page?: number;
    @IsOptional() @Type(() => Number) @IsInt() @Min(1) limit?: number;
}
```

```ts
// src/components/marketplacev2/order/enum/domain-listing-status.enum.ts
export const DOMAIN_LISTING_STATUS_VALUES = ['LISTED', 'UNLISTED', 'SOLD'] as const;
export type DomainListingStatus = (typeof DOMAIN_LISTING_STATUS_VALUES)[number];

export const MY_DOMAINS_STATUS_FILTER_VALUES = [...DOMAIN_LISTING_STATUS_VALUES, 'ALL'] as const;
export type MyDomainsStatusFilter = (typeof MY_DOMAINS_STATUS_FILTER_VALUES)[number];
```

`status` accepts `LISTED`, `UNLISTED`, `SOLD`, or `ALL` (the same as omitting it). `expiryStatus` filters on the domain's real world registration expiry and combines with `status` as an AND. The frontend drives a two column layout (listed on one side, unlisted on the other) by making two independent calls with different `status` values and separate page state, exactly like the v1 marketplace's columns.

The service is where "my domains" reaches outside the marketplace module:

```ts
// src/components/marketplacev2/order/order.service.ts
async myDomains(userId: string, query: MyDomainsQuery): Promise<ResultResponse> {
    const page = query.page ?? 1;
    const limit = query.limit ?? 10;

    const [domains, statuses, soldAtByKey] = await Promise.all([this.domainDetailBCRepo.findById(userId), this.listingStatusService.findAllForUser(userId), this.listingHistoryService.findSoldForUser(userId)]);

    const statusByKey = new Map(statuses.map((row) => [`${row.domainName}::${row.tokenId}`, row.status]));
    let domainsWithStatus = domains.map((domain) => {
        const key = `${domain.domainName}::${domain.token_id}`;
        const soldAt = soldAtByKey.get(key);
        const status: DomainListingStatus = soldAt !== undefined && soldAt > domain.createdDateTime ? 'SOLD' : statusByKey.get(key) ?? ListingStatus.UNLISTED;
        return {
            domain,
            status,
            expiryStatus: classifyDomainExpiry(domain.domainProvider, domain.expiryDate)
        };
    });
    if (query.status && query.status !== 'ALL') {
        domainsWithStatus = domainsWithStatus.filter((entry) => entry.status === query.status);
    }
    if (query.expiryStatus) {
        domainsWithStatus = domainsWithStatus.filter((entry) => entry.expiryStatus === query.expiryStatus);
    }

    const total = domainsWithStatus.length;
    const skip = getPaginationSkipByPageAndLimit(page, limit);
    const pageEntries = domainsWithStatus.slice(skip, skip + limit);
    // ... then batched order lookups for the page only, see below
}
```

Three sources are loaded in parallel. The first is `tbl_domain_detail_bc` via `DomainDetailBCRepoInterface.findById(userId)`, which is the old domain detail module's cache of every NFT domain the user's wallets own across every provider. That table is populated by `DomainDetailService.refreshDomainDetailData`, which calls the Alchemy based per chain services (Freename, Arbitrum, BNB, UD Base, ENS, UD) for the user's EVM wallet and Moralis for Solana. So "my domains" in v2 is really "the domain detail module's last known view of my wallets", annotated with marketplace state; it never calls Alchemy itself on a read. The second source is `tbl_marketplacev2_listing_status` (the current LISTED or UNLISTED per domain, see `07`), and the third is the SOLD events from `tbl_marketplacev2_listing_history`, reduced to the newest `occurredAt` per `domainName::tokenId`.

The status rule is the subtle part. A fill flips the listing status row to UNLISTED, which would make a sold domain look the same as one never listed, so SOLD takes priority, but only if the sale happened after the domain's `createdDateTime`. That timestamp comes from `BaseEntity` and is effectively "the last time a refresh confirmed you own this", because the refresh deletes and reinserts the user's rows. If the sale is newer than the last confirmation, the domain shows SOLD; if a refresh has confirmed ownership since (the user bought it back elsewhere), it falls through to the normal status. The `order.service.my-domains.spec.ts` file has a regression test named for exactly this reported bug. There is a consequence worth spelling out for a frontend developer, though: since `findById` reads only the domains the user currently owns per the last refresh, a sold domain is only visible as SOLD until the next refresh, at which point the domain detail module drops the row entirely and it disappears from my domains. So `status=SOLD` effectively means "sold since my last sync", not "everything I have ever sold"; for the permanent record, use `listing-history` or `transactions`.

Filtering and pagination happen in memory over the full list. Only after the page slice is taken does the method fetch order details, and only for the rows on that page:

```ts
// src/components/marketplacev2/order/order.service.ts
const listedTokenIdsOnPage = pageEntries.filter((entry) => entry.status === ListingStatus.LISTED && entry.domain.token_id).map((entry) => entry.domain.token_id as string);
const soldTokenIdsOnPage = pageEntries.filter((entry) => entry.status === 'SOLD' && entry.domain.token_id).map((entry) => entry.domain.token_id as string);

const [liveOrders, filledOrders] = await Promise.all([
    listedTokenIdsOnPage.length
        ? (() => {
            const qb = this.orderRepository.createQueryBuilder('o').andWhere('o.tokenContract = :tokenContract', { tokenContract: this.chainConfig.domainNftAddress }).andWhere('o.tokenId IN (:...tokenIds)', { tokenIds: listedTokenIdsOnPage });
            applyServableWhere(qb, 'o');
            return qb.getMany();
        })()
        : Promise.resolve([]),
    soldTokenIdsOnPage.length ? this.orderRepository.find({ where: { tokenContract: this.chainConfig.domainNftAddress, tokenId: In(soldTokenIdsOnPage), status: OrderStatus.FILLED } }) : Promise.resolve([])
]);
```

That is two batched queries per page, never one per row, so there is no N+1 here. The `IN (:...tokenIds)` spread syntax is TypeORM's way of expanding an array into a parameter list. Each returned row then looks like this:

| Field | Source | Notes |
|---|---|---|
| `domainName` | `tbl_domain_detail_bc.domainName` | |
| `tld` | `domainName.split('.').pop()` | last label only |
| `tokenId` | `token_id` or `''` | |
| `domainProvider` | `domainProvider` | e.g. `ENS`, `Arbitrum` |
| `status` | `LISTED`, `UNLISTED`, or `SOLD` | rule above |
| `order` | live order (if LISTED) or filled order (if SOLD), else `null` | `orderHash`, `priceUsdt`, `feeUsdt`, `sellerUsdt`, `startTime`, `endTime`, `createdAt`, `filledBy`, `filledAt` |
| `expiryDate` | `expiryDate` or `null` | Unix seconds string |
| `expiryStatus` | `classifyDomainExpiry` | `Expired`, `GracePeriod`, `ExpiringSoon`, `Normal` |

`classifyDomainExpiry` (`util/domain-expiry-status.util.ts`) only treats `ENS`, `BinanceSmartChain`, and `Arbitrum` as expirable, needs a purely numeric `expiryDate`, and buckets by "more than 90 days past expiry is Expired, past expiry but within 90 days is GracePeriod, within the next 30 days is ExpiringSoon, otherwise Normal". The comment notes it is kept in sync by hand with a SQL twin used by my listings.

The order of the list is whatever `findById` returns, which is `ORDER BY domainName DESC` (Z to A). A LISTED domain whose order has since passed its `endTime` (but the sweep has not yet run) will show LISTED with `order: null`, which the spec explicitly accepts as the intended rendering.

### `GET /my-listings`: order history as the seller

```ts
// src/components/marketplacev2/order/enum/listing-category-filter.enum.ts
export const LISTING_CATEGORY_FILTERS = ['active', 'sold', 'cancelled', 'expired', 'expiring_soon'] as const;
export type ListingCategoryFilter = (typeof LISTING_CATEGORY_FILTERS)[number];
```

```ts
// src/components/marketplacev2/order/dto/my-listings-query.dto.ts
export class MyListingsQueryDto {
    @IsOptional() @IsIn(LISTING_CATEGORY_FILTERS) category?: ListingCategoryFilter;
    @IsOptional() @Type(() => Number) @IsInt() @Min(1) page?: number;
    @IsOptional() @Type(() => Number) @IsInt() @Min(1) limit?: number;
}
```

Where my domains starts from ownership, my listings starts from the orders table and returns every order the caller's verified wallet has ever made, any status, newest first. The category filters map to `WHERE` clauses through a lookup table on the service:

```ts
// src/components/marketplacev2/order/order.service.ts
private static readonly CATEGORY_FILTERS: Record<ListingCategoryFilter, (qb: SelectQueryBuilder<OrderEntity>, expiryStatusExpr: string) => void> = {
    active: (qb) => qb.andWhere('o.status = :activeStatus', { activeStatus: OrderStatus.ACTIVE }),
    sold: (qb) => qb.andWhere('o.status = :filledStatus', { filledStatus: OrderStatus.FILLED }),
    cancelled: (qb) => qb.andWhere('o.status = :cancelledStatus', { cancelledStatus: OrderStatus.CANCELLED }),
    expired: (qb, expiryStatusExpr) => qb.andWhere(`${expiryStatusExpr} = 'Expired'`),
    expiring_soon: (qb, expiryStatusExpr) => qb.andWhere(`${expiryStatusExpr} = 'ExpiringSoon'`)
};
```

Typing it as `Record<ListingCategoryFilter, ...>` means TypeScript refuses to compile if someone adds a value to `LISTING_CATEGORY_FILTERS` without adding its clause, a lovely use of the type system as a checklist. The query itself:

```ts
// src/components/marketplacev2/order/order.service.ts
async myListings(walletAddress: string, userId: string, query: MyListingsQuery): Promise<ResultResponse> {
    const page = query.page ?? 1;
    const limit = query.limit ?? 10;
    const skip = getPaginationSkipByPageAndLimit(page, limit);
    const maker = getAddress(walletAddress);
    const expiryStatusExpr = expiryStatusCaseSql('bc');
    const qb = this.orderRepository
        .createQueryBuilder('o')
        .leftJoin((sub) => sub.select('bc.*').from(DomainDetailBCEntity, 'bc').where('bc."userId" = :userId', { userId }), 'bc', 'LOWER(bc."domainName") = LOWER(o."domainName")')
        .where('o.maker = :maker', { maker })
        .select([
            'o."orderHash" AS "orderHash"',
            // ... domainName, tld, tokenId, status, priceUsdt, feeUsdt, sellerUsdt,
            // startTime, endTime, createdAt, filledAt, filledBy, invalidReason
            'bc."expiryDate" AS "expiryDate"',
            `${expiryStatusExpr} AS "expiryStatus"`
        ]);

    const categoryFilter = query.category && OrderService.CATEGORY_FILTERS[query.category];
    if (categoryFilter) {
        categoryFilter(qb, expiryStatusExpr);
    }

    const total = await qb.getCount();

    qb.orderBy('o."createdAt"', 'DESC').limit(limit).offset(skip);
    const data = await qb.getRawMany<MyListingResult>();

    return paginateResponse(data, total, new PaginationRequestDto(limit, skip, page));
}
```

The `maker` is checksummed with `getAddress` because `OrderEntity.maker` is stored checksummed, so a plain `=` uses `idx_marketplacev2_orders_maker`. The left join is to a subquery of `tbl_domain_detail_bc` scoped to this user, matched on `LOWER(domainName)`, purely to attach `expiryDate` and compute `expiryStatus` in SQL via `expiryStatusCaseSql('bc')` (`util/expiry-status.util.ts`), which is a `CASE` expression with the same providers and thresholds as the JS classifier. A listing with no matching domain row comes back with `expiryDate: null` and `expiryStatus: 'Normal'`, never excluded. `getRawMany` is used because the select list is hand built with aliases; the rows come back as plain objects shaped like `MyListingResult` (`orderHash`, `domainName`, `tld`, `tokenId`, `status`, `priceUsdt`, `feeUsdt`, `sellerUsdt`, `startTime`, `endTime`, `createdAt`, `filledAt`, `filledBy`, `invalidReason`, `expiryDate`, `expiryStatus`).

Two inconsistencies in the filter set deserve attention. First, `expired` means the domain's registration expired (from `tbl_domain_detail_bc`), not that the listing expired, even though `OrderStatus.EXPIRED` exists and `expireIfStale` writes it. A seller looking for "my listings that timed out" has no filter for that, and no filter for `invalid` either. Second, the SQL classifier can also yield `GracePeriod`, but there is no `grace_period` filter value. Neither is a crash, both are product gaps a frontend dropdown will expose.

### `GET /my-purchases`: order history as the buyer

```ts
// src/components/marketplacev2/order/order.service.ts
async myPurchases(walletAddress: string, query: MyPurchasesQuery): Promise<ResultResponse> {
    const page = query.page ?? 1;
    const limit = query.limit ?? 10;
    const skip = getPaginationSkipByPageAndLimit(page, limit);

    const qb = this.orderRepository
        .createQueryBuilder('o')
        .where('LOWER(o."filledBy") = LOWER(:buyer)', { buyer: walletAddress })
        .andWhere('o.status = :status', { status: OrderStatus.FILLED });

    const total = await qb.getCount();
    const orders = await qb.orderBy('o."filledAt"', 'DESC').limit(limit).offset(skip).getMany();

    const data: MyPurchaseResult[] = orders.map((order) => ({
        orderHash: order.orderHash,
        domainName: order.domainName,
        tld: order.tld,
        tokenId: order.tokenId,
        seller: order.maker,
        priceUsdt: order.priceUsdt,
        feeUsdt: order.feeUsdt,
        sellerUsdt: order.sellerUsdt,
        filledAt: order.filledAt,
        fillTxHash: order.fillTxHash,
        createdAt: order.createdAt
    }));

    return paginateResponse(data, total, new PaginationRequestDto(limit, skip, page));
}
```

The DTO is just `page` and `limit`. The query compares `filledBy` case insensitively because it is written straight from Seaport's `OrderFulfilled.recipient`, and the code does not want to bet on its casing matching the caller's stored wallet. The price of `LOWER(...)` is that no index can serve it (there is no index on `filledBy` at all), though the `status = 'filled'` condition can use `idx_marketplacev2_orders_status` to narrow first. `GET /transactions/mine`, covered in `07`, is the receipt table version of this same view, adding `txHash`, `blockNumber`, and gas fields.

### `GET /listing-history`: briefly

The DTO accepts optional `domainName` and `tokenId` (both `@IsString`, combined with AND) plus the usual `page`/`limit`. `OrderService.listingHistory` delegates to `ListingHistoryService.findForUser`, maps rows to `ListingHistoryEntryDto` (`orderHash`, `domainName`, `tokenId`, `tokenContract`, `eventType`, `txHash`, `occurredAt`, deliberately never `userId` or `createdAt`), and wraps the result with `paginateResponse`. The table and its writers are explained in `07`.

## Cursor pagination versus offset pagination, properly

Now that you have seen both styles in real code, here is the conceptual comparison, because it is one of the most valuable things a frontend developer can absorb on the way to fullstack.

Offset pagination says "skip the first N rows of this ordering, then give me the next L". It is what `paginateResponse` and `getPaginationSkipByPageAndLimit` implement everywhere else in this codebase. Its strengths are real: you can jump straight to page 7, you can show "page 3 of 12", and you get a `total`. Its weaknesses all come from the fact that "skip N" is relative to whatever the table looks like at the moment of the request. If a new listing is inserted at the top while the user is reading page 1, every row shifts down one position, so the last row of page 1 is now the first row of page 2, and the user sees it twice. If a listing on page 1 sells and disappears, every row shifts up one, and the first row that would have been on page 2 is now on page 1, which the user has already loaded, so they never see it. On a busy public marketplace with an infinite scroll, both happen constantly. Offset also gets slower the deeper you go, because Postgres must actually walk and discard the first N rows, and it needs a separate `COUNT(*)` for the total, which on a big filtered set is often as expensive as the page itself.

Cursor (also called keyset) pagination says "give me the next L rows strictly after this exact position in the ordering". The position is the last row you saw, identified by its sort key plus a unique tie breaker. Insertions above that position and deletions below it no longer shift anything, because the query is anchored to a value, not a count. A sold listing simply is not in the next page; a newly created one appears only if it sorts after your position. With a matching index, Postgres can seek straight to the position, so page 500 costs the same as page 1. The tradeoffs are that you cannot jump to an arbitrary page, there is no natural total (hence `categoryCounts` being a separate, first page only computation), and the cursor is only meaningful for the exact sort and filter set that produced it.

Which is why the split in this module is sensible. Browse is public, high churn, sorted, and consumed by infinite scroll, which is the textbook case for cursors. The "my" pages are small, private, low churn tables where a user expects page numbers and totals, and where a concurrent writer racing a scroll is rare, so offset is the pragmatic choice; `ListingHistoryService.findForUser`'s own comment says as much. The public recent sales feed is a bounded "latest few" widget, so it uses the simplest possible shape. The only real criticism is that this produces three different response envelopes for a frontend to handle (`{ items, nextCursor }`, `{ data, paginationInfo }`, and `{ data, total }`), which is worth wrapping in a small client side adapter.

## Consuming browse from React with infinite scroll

The intended client contract is short: send the filters, get `items` and `nextCursor`; if `nextCursor` is not `null`, send exactly the same filters plus `cursor=nextCursor` for the next page; when it is `null`, stop. Never construct, decode, or modify a cursor, and throw away all loaded pages and cursors whenever any filter, the sort, or the tab changes. Read `categoryCounts` from the first page only, since later pages omit it. With TanStack Query that maps almost one to one onto `useInfiniteQuery`:

```ts
// example frontend hook (illustrative, not in this repo)
import { useInfiniteQuery } from '@tanstack/react-query';

type BrowseFilters = { tld?: string; maxPrice?: string; q?: string; sort?: 'recent' | 'price_asc' | 'price_desc'; category?: string; limit?: number };

async function fetchBrowsePage(filters: BrowseFilters, cursor: string | null) {
    const params = new URLSearchParams();
    Object.entries(filters).forEach(([key, value]) => value !== undefined && value !== '' && params.set(key, String(value)));
    if (cursor) params.set('cursor', cursor);
    const res = await fetch(`/api/v1/marketplacev2/orders?${params.toString()}`);
    if (!res.ok) throw new Error(`browse failed: ${res.status}`);
    const body = await res.json();
    return body.result as { items: OrderSummary[]; nextCursor: string | null; categoryCounts?: Record<string, number> };
}

export function useBrowseOrders(filters: BrowseFilters) {
    return useInfiniteQuery({
        // filters in the key: any change starts a brand new list with no cursor
        queryKey: ['marketplacev2-browse', filters],
        queryFn: ({ pageParam }) => fetchBrowsePage(filters, pageParam),
        initialPageParam: null as string | null,
        getNextPageParam: (lastPage) => lastPage.nextCursor ?? undefined
    });
}

// usage: const q = useBrowseOrders(filters);
// const items = q.data?.pages.flatMap((p) => p.items) ?? [];
// const counts = q.data?.pages[0]?.categoryCounts;
// an IntersectionObserver sentinel calls q.fetchNextPage() when q.hasNextPage && !q.isFetchingNextPage
```

Putting `filters` inside the `queryKey` is what enforces "new filters, new list", since TanStack Query treats a different key as a different cache entry with its own page chain. A few practical notes follow from the backend code. Debounce the `q` input (300 ms or so), because every keystroke is a new first page with six queries behind it. Deduplicate by `orderHash` when flattening pages as a cheap safety net. Expect the first page to be up to 60 seconds stale, since `Cache-Control: public, max-age=60` lets the browser and any CDN reuse it; later pages carry a cursor in the URL so they are distinct cache entries, but they can also be reused for 60 seconds. When a user clicks a card to buy, fetch `GET /:orderHash` fresh rather than trusting the browse item, since the item may be a minute old and the detail route is the one that decides whether a signature is still served. And for the "my" pages, use ordinary numbered pagination driven by `paginationInfo.nextPage` and `lastPage`, or `useInfiniteQuery` with `getNextPageParam: (last) => last.paginationInfo.nextPage ?? undefined` if you want a scroll UX there too.

## Spec files covering this read side

`src/components/marketplacev2/order/util/order-cursor.util.spec.ts` has six tests. It round trips a cursor through `encodeCursor` then `decodeCursor` and expects deep equality; it rejects a string that is not base64url JSON at all; it rejects valid base64url whose content is not JSON; it rejects JSON missing `orderHash`; it rejects a malformed `orderHash` (`'not-a-hash'`); and it rejects a numeric `sortKey`. Every rejection is asserted as a `BadRequestException`. What it does not test is a semantically wrong but well typed `sortKey`, such as an ISO date used with a price sort, which is exactly the gap that becomes a 500 below.

`src/components/marketplacev2/order/util/domain-category.util.spec.ts` covers `extractLabel` (strips `.tld` using the column, falls back to the whole name when the suffix does not match), `isShortLabel` (true for 2 and 3, false for 1 and 4), `isNumericLabel` (a digit anywhere counts, none means false), `isPremiumTld` (always false while the list is empty), `isDictionaryLabel` (exact match, case insensitive for `flower`/`FLOWER`, false for the made up `zibbexqor`, false for `flowerpotxyz` because substring matches do not count), `classifyDomainCategories` (`go.com` is short and dictionary but not brandable; `a1.com` is short and numeric; `zibbexqor.com` is brandable only; `elephant.com` is dictionary and not brandable), and `buildSqlCategoryWhere` (`null` for `all`, contains `IN (2, 3)` for short, contains `~ '[0-9]'` for numeric, and exactly `LOWER(o."tld") = ANY(ARRAY[]::text[])` for premium). Because the dictionary tests actually call `require('an-array-of-english-words')`, this spec is also the smoke test for that dependency.

`src/components/marketplacev2/order/order.service.spec.ts` builds an `OrderService` with hand rolled mocks (a `chainableQb` fake whose `andWhere`, `orderBy`, `limit` and friends all return itself and whose `getMany`/`getCount`/`getRawOne` resolve to fixed values, plus a `makeOrderRow` fixture), so it tests the service's own logic, not the SQL text. In its `browse()` block, three fixture rows (`flower.com`, `zibbex.com`, `ab.com`) back five tests: with `limit: 2` and three rows returned, the SQL path returns two items and a `nextCursor` that decodes to the second row's hash and ISO `createdAt`, plus counts of 5 for each SQL tab and JS computed dictionary and brandable counts; with only one row returned, `nextCursor` is `null`; the `dictionary` and `brandable` tabs return exactly the rows the classifier flags; a request carrying a cursor has `categoryCounts` undefined; and a malformed cursor rejects with `BadRequestException`. Its `stats()` block checks all zeroes and `null` median on an empty table, correct mapping of live count, volume, sales count, median string, and `lastIndexedBlock` from the poller cursor, and a default of 0 when the cursor row does not exist. Its `listingHistory()` block (TC6.8, TC6.9) checks the DTO mapping never leaks `userId`, `id`, or `createdAt`, that `domainName`/`tokenId` pass through to the repository, that page and limit default to 1 and 10, and that an empty history is an empty paginated result. The same file also covers the health endpoints, `activeListingsHealth` mapping, and `findByHash` throwing `NotFoundException`.

`src/components/marketplacev2/order/order.service.my-domains.spec.ts` builds the service with a stubbed `findById`, `findAllForUser`, `findSoldForUser`, and a query builder whose `getMany` is spied on. It verifies that a domain with no status row defaults to UNLISTED with `order: null`; that a LISTED domain gets its live order summary via exactly one batched `getMany` call, with the full documented row shape including `expiryDate: null` and `expiryStatus: 'Normal'`; that LISTED with no matching live order still renders LISTED with `order: null`; and that an empty domain list is an empty page. Its status filter block checks that `LISTED` excludes unlisted rows from both data and count and that `UNLISTED` never queries for live orders at all. Its pagination block checks page size and that `paginationInfo` reflects the filtered total, and that page 2 returns the next slice with `prevPage: 1`. Its SOLD block holds the two regression tests: sold after the last sync renders SOLD with the filled order attached, and sold then reacquired and resynced renders UNLISTED with no order.

## Bugs, risks, and inconsistencies

**A cursor from one sort used with another crashes with a 500.** `decodeCursor` checks that `sortKey` is a string but not that it matches the sort (`order-cursor.util.ts:38`). If a frontend keeps its cursor state when the user flips the sort dropdown from `recent` to `price_asc`, the request carries an ISO date as `sortKey`; on the SQL path `'2026-09-30T12:00:00.000Z'::bigint` fails in Postgres (`order.service.ts:1115`), and on the dictionary or brandable path `BigInt(cursor.sortKey)` throws a `SyntaxError` (`order.service.ts:1202`). Either way the user sees a 500 instead of a 400. The same applies to a hand edited cursor with a garbage `sortKey` on `recent` (`order.service.ts:1128`). Encoding `sort` inside the cursor and validating it on decode would fix both.

**The `recent` cursor depends on the Node process time zone.** `createdAt` is a `timestamp` (without time zone) column written as `new Date()` (`order.service.ts:681`, entity `order.entity.ts:96`). The node postgres driver writes and reads such columns as local wall clock time, but the cursor is minted with `toISOString()` (UTC) and compared via `:cursorCreatedAt::timestamp`, which discards the `Z` (`order.service.ts:1128`, `order.service.ts:1135`). On a UTC server this is consistent; on a server or developer machine set to, say, IST, the cursor is five and a half hours off from the stored values, so page two either repeats a large chunk of page one or skips it. Worth verifying that every environment runs with `TZ=UTC`, or switching the column to `timestamptz`.

**Dictionary and brandable silently truncate past 5,000 rows.** `scanServableRows` caps the scan at `DICTIONARY_SCAN_CAP` (`order.service.ts:1183`), so once more than 5,000 listings match the base filters, matching listings beyond the cap never appear on those tabs, the scroll ends with `nextCursor: null`, and the tab counts (`order.service.ts:1237`) undercount while the SQL counts stay exact. Every in app page also rescans the full 5,000 rows.

**First page browse runs six queries.** `computeCategoryCounts` always runs four `COUNT(*)` queries plus a 5,000 row scan even on the `all` tab (`order.service.ts:1224`), and nothing caches it server side beyond the 60 second HTTP header.

**`medianAsk` can be fractional.** `PERCENTILE_CONT` interpolates (`order.service.ts:1065`), so an even number of live listings can yield `"150.5"`, which breaks `BigInt` parsing in any client following the module's integer minor unit convention.

**`maxPrice` accepts decimals.** `@IsNumberString()` (`order-browse-query.dto.ts:19`) lets `maxPrice=1.5` through, and `:maxPrice::bigint` then throws a 500 instead of a validation 400.

**No `@Max` on any "my" route's `limit`.** `my-domains-query.dto.ts:31`, `my-listings-query.dto.ts:17`, `my-purchases-query.dto.ts:12`, `my-transactions-query.dto.ts:12`, and `listing-history-query.dto.ts:21` all accept unbounded limits, unlike browse's 100. For my domains in particular a huge limit makes the page's token id list unbounded, and a page with more than roughly 65,000 listed token ids would exceed Postgres's bind parameter limit in the `IN (:...tokenIds)` query (`order.service.ts:851`).

**My domains loads everything into memory on every page.** `findById`, `findAllForUser`, and `findSoldForUser` are all unbounded per user reads (`order.service.ts:807`), filtered and sliced in JS. Fine for typical wallets, linear for whales. `findById` (`domain-detail-bc.repo.ts:181`) also does not filter `isDeleted`, so any soft deleted rows would be shown and counted.

**My domains key matching is case sensitive across modules.** The status map key is `domainName::tokenId` from `tbl_marketplacev2_listing_status` and the lookup key is built from `tbl_domain_detail_bc` (`order.service.ts:809`, `order.service.ts:811`). If the two tables ever store different casing for the same name, a LISTED domain renders UNLISTED. My listings, by contrast, joins with `LOWER(...)` on both sides.

**SOLD in my domains is transient and the filled order pick is arbitrary.** A sold domain disappears after the seller's next refresh, since `findById` reflects current ownership only (`order.service.ts:807`). And if the same token was sold through the marketplace more than once, the `find` at `order.service.ts:856` has no `order`, and the `Map` keeps whichever filled order happens to come last, so the attached order may be an older sale.

**My listings can return duplicate rows.** The left join matches `tbl_domain_detail_bc` on `LOWER(domainName)` only (`order.service.ts:909`), while that table's unique key is `(domainName, token_id, userId)`. If a user has two rows with the same name and different token ids (a remint, or the same name tracked by two providers), each order row is returned twice by `getRawMany`, while `getCount` (which counts distinct primary keys) reports the true number, so pages contain duplicates and the page math is off. The subquery also does not filter `isDeleted`.

**My listings' `expired` means domain expiry, not listing expiry.** `CATEGORY_FILTERS.expired` (`order.service.ts:173`) filters on the registration CASE expression, and there are no filters for `OrderStatus.EXPIRED` or `OrderStatus.INVALID`, nor for `GracePeriod`.

**My purchases cannot use an index.** `LOWER(o."filledBy") = LOWER(:buyer)` (`order.service.ts:957`) has no supporting functional index; the query narrows by `status` and then scans filled rows.

**"My" views follow only the currently verified wallet.** My listings, my purchases, and transactions mine all scope by `req.walletAddress`, so a user who verified a different wallet last month sees none of that wallet's history, and the user must re sign every 60 minutes just to read their own purchase history, while my domains and the watchlist need only a JWT.

**Business logic in the controller.** `recentTransactions` and `myTransactions` compute defaults, skip, and pagination in the controller (`order.controller.ts:170`, `order.controller.ts:193`), against the project's thin controller convention, and inject `TransactionService` by class rather than by interface token like every other service here.

**Three pagination envelopes.** Browse returns `{ items, nextCursor, categoryCounts? }`, the "my" routes return `{ data, paginationInfo }`, and transactions recent returns `{ data, total }`; not a bug, but a frontend needs three adapters.

**`OrderQueryDto` is dead code.** `dto/order-query.dto.ts` is described in its own comment as an unconfirmed assumption for a future browse endpoint and is not used by any route; the real browse uses `OrderBrowseQueryDto`.

**`premium` builds SQL by string interpolation.** `buildSqlCategoryWhere` interpolates `PREMIUM_TLDS` into the SQL text (`domain-category.util.ts:129`). Safe while the list is a hardcoded constant, but it becomes an injection vector the day the list is made configurable.

## Frontend note

If you have ever built a product listing page with "Load more", you have consumed a cursor without thinking about it; this file is what that looks like from the other side. The lessons worth carrying forward are that the cursor's `WHERE` must mirror the `ORDER BY` exactly, that a unique tie breaker is what makes the ordering safe to page through, that "fetch one extra row" replaces a `COUNT(*)`, and that a cursor is only valid for the exact filters and sort that produced it, so the client must reset whenever those change. The parts of this code that are weakest are the parts where that last rule is not enforced by the server, a good reminder that an "opaque" token is only as safe as the validation done when it comes back.
