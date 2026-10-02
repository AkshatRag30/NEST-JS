# 03. A Seaport Primer and the Order Entity

## Why marketplace v2 exists at all, and why it looks nothing like v1

If you have read `03-commerce-and-marketplace/03-domain-listing-and-buy-domain-secondary-marketplace.md`, you already know the old resale flow. In v1, a seller sends a real `createListing` transaction to Endless Domains' own marketplace contract, pays gas for it, and the backend sits around waiting for a cron job to confirm that transaction landed. Every listing costs the seller money and time before a single buyer has even seen it, and every cancellation costs gas again.

Marketplace v2, which lives in `src/components/marketplacev2/`, throws that model away and replaces it with the one OpenSea, Blur and most modern NFT marketplaces use: Seaport style signed orders. Listing becomes free and instant. The seller does not send a transaction at all. Instead, their wallet signs a structured message that says, in effect, "I, this wallet, promise that anyone who pays these exact amounts to these exact recipients, between this start time and this end time, may take this exact NFT from me." The backend stores that signed promise in a Postgres table. Later, a buyer fetches it, hands it to the Seaport smart contract along with their payment, and Seaport itself checks the signature and swaps the NFT for the money atomically in one transaction. Nothing is on chain until the moment of sale.

All of the v2 code landed after the baseline commit `a131b429` the rest of these notes were written against, starting with `836d5f89` ("Added new marketpalce v2 code", 10 September 2026) and running through `bdd8e80a` ("Added new changes in the marketplacev2", 1 October 2026). The sprint labels you will see sprinkled through the code comments (`B-02`, `B-03 Sprint 2`, `B-04` and so on) refer to the internal plan documents (`B03_IMPLEMENTATION_PROMPT.md`, `B04_MASTER_PLAN.md`, `B05_B08_MASTER_PLAN.md`) that are not in this repository; the comments quote from them often enough that you can reconstruct the intent.

This file is the conceptual foundation for the rest of the cluster. It explains what a Seaport order is, piece by piece, then walks the `tbl_marketplacev2_orders` table column by column, then explains the shared `@endlessdomains/order-builder` package and the idea of "parity" between frontend and backend. File `04` covers how an order is created and validated, and file `05` covers how one is cancelled or expires.

## Seaport in one paragraph

Seaport is a general purpose, audited marketplace protocol originally written by OpenSea and deployed at the same address on every EVM chain. This app uses Seaport version 1.6 on Polygon mainnet (chain id `137`), at `0x0000000000000068F116a894984e2DB1123eB395`, an address recorded in the header comment of `order/abi/chain-check.abi.ts`. Seaport's mental model is beautifully simple once it clicks: every order has an offer side (what the order's creator is giving up) and a consideration side (what the order's creator, and possibly other people, must receive in return). Whoever fills the order supplies the consideration and receives the offer. That one abstraction covers listings, bids, bundles, collection offers and royalties, which is exactly why it has become the industry standard.

## The pieces of an order, one at a time

The structure the seller actually signs is called `OrderComponents`. The frontend's ethers v5 mirror of the shared package (in the separate frontend repo, `src/lib/seaport-order-builder-v5.ts`) spells out its EIP712 type definition exactly, and that type list is the single most important thing in this whole cluster, because the field order is the hash:

```ts
// endlessdomains-marketplace-frontend/src/lib/seaport-order-builder-v5.ts
export const EIP712_TYPES = {
    OrderComponents: [
        { name: 'offerer', type: 'address' },
        { name: 'zone', type: 'address' },
        { name: 'offer', type: 'OfferItem[]' },
        { name: 'consideration', type: 'ConsiderationItem[]' },
        { name: 'orderType', type: 'uint8' },
        { name: 'startTime', type: 'uint256' },
        { name: 'endTime', type: 'uint256' },
        { name: 'zoneHash', type: 'bytes32' },
        { name: 'salt', type: 'uint256' },
        { name: 'conduitKey', type: 'bytes32' },
        { name: 'counter', type: 'uint256' }
    ],
    OfferItem: [
        { name: 'itemType', type: 'uint8' },
        { name: 'token', type: 'address' },
        { name: 'identifierOrCriteria', type: 'uint256' },
        { name: 'startAmount', type: 'uint256' },
        { name: 'endAmount', type: 'uint256' }
    ],
    ConsiderationItem: [
        { name: 'itemType', type: 'uint8' },
        { name: 'token', type: 'address' },
        { name: 'identifierOrCriteria', type: 'uint256' },
        { name: 'startAmount', type: 'uint256' },
        { name: 'endAmount', type: 'uint256' },
        { name: 'recipient', type: 'address' }
    ]
} as const;
```

Here is what every one of those fields means in plain language, and what value Endless Domains always puts in it.

`offerer` is the seller's wallet address, the one whose signature must authorise the order and whose NFT will move. The backend insists it equals the caller's verified wallet (check 1 in file `04`).

`offer` is the list of things the offerer gives up. For a domain listing it is always exactly one item: `itemType` `2` (ERC721), `token` set to the configured domain NFT contract, `identifierOrCriteria` set to the domain's token id, and `startAmount` and `endAmount` both `1`, because you cannot sell half an NFT.

`consideration` is the list of payments the filler must make. Endless Domains always uses exactly two legs, both `itemType` `1` (ERC20) in the configured USDT contract. Leg zero pays the seller their share and has `recipient` set to the offerer. Leg one pays the marketplace fee and has `recipient` set to the configured fee recipient. Each leg's `startAmount` and `endAmount` must be equal; Seaport supports linear price curves (a Dutch auction is just an order whose amounts change between start and end time), but this marketplace only sells at a flat price. `identifierOrCriteria` is `0` for an ERC20 leg, because fungible tokens have no individual ids.

`orderType` encodes two independent yes or no questions in one small integer: can the order be partially filled, and is it restricted to a zone. The values are `FULL_OPEN` `0`, `PARTIAL_OPEN` `1`, `FULL_RESTRICTED` `2` (with Seaport itself also defining `PARTIAL_RESTRICTED` `3` and `CONTRACT` `4`). A single NFT can never be partially filled, and this marketplace has no need for a gatekeeper contract, so every order must be `FULL_OPEN`.

`zone` and `zoneHash` are the restricted order machinery. A zone is a contract Seaport consults before allowing a restricted order to fill (useful for things like "only allow this fill if the buyer passed KYC"), and `zoneHash` is an arbitrary 32 byte value passed to that zone. Because orders here are `FULL_OPEN`, `zone` must be the zero address and `zoneHash` must be 32 zero bytes.

`startTime` and `endTime` are Unix timestamps in seconds. Seaport refuses to fill an order when the block's timestamp is before `startTime` or at or after `endTime`. The shared builder backdates `startTime` by 300 seconds at signing (`START_TIME_BACKDATE_SECONDS`), so that a block whose clock is a little behind the seller's laptop still accepts the order immediately, and it defaults the window to seven days (`ORDER_DURATION_SECONDS`). The backend accepts anything from one hour to ninety days of remaining life.

`salt` is just a random 256 bit number. Its only job is to make two otherwise identical orders produce different hashes, so a seller who lists the same domain at the same price twice gets two distinct orders instead of a collision. The builder fills it with 32 random bytes.

`conduitKey` selects a conduit, Seaport's optional approval proxy. Conduits let a user approve one conduit contract once and have it move tokens on behalf of several marketplaces. Endless Domains deliberately does not use one: `conduitKey` must be 32 zero bytes (`NO_CONDUIT_KEY`), which tells Seaport "transfer the tokens yourself," and it means the seller's approval must be given to the Seaport contract directly via `setApprovalForAll`. That is why check 10 in file `04` asks `isApprovedForAll(offerer, seaportAddress)` rather than asking about a conduit.

`counter` is a per offerer nonce stored inside Seaport. Every order signed by an offerer embeds that offerer's current counter. If the offerer ever calls `incrementCounter()` on Seaport, every order they ever signed against the old value becomes invalid in one transaction. It is Seaport's "cancel everything" button, and it is why the backend reads the live counter at creation time (check 8, `STALE_COUNTER`).

## Order hash, digest and signature: three different 32 byte things

Beginners often blur these together, and the backend code is careful to keep them apart, so it is worth slowing down.

The order hash is the EIP712 `hashStruct` of the `OrderComponents` object: the struct is encoded field by field in the exact type order above, each nested array of items is hashed too, and the whole thing is run through keccak256. Seaport itself computes this same value on chain (it is what `getOrderHash` returns and what `getOrderStatus(orderHash)` looks up), so the order hash is the order's permanent public identity. The backend computes it with `computeOrderHash(components)` and uses it as the table's primary key. It is never accepted from the client; the entity's own comment says so: `bytes32 hex. Computed server-side in B-03, never accepted from the client.`

The digest is what actually gets signed. EIP712 wraps the order hash with a domain separator, which binds the signature to one particular contract on one particular chain: digest equals keccak256 of the bytes `0x19 0x01`, followed by the domain separator, followed by the order hash. The domain here is `{ name: 'Seaport', version: '1.6', chainId: 137, verifyingContract: <Seaport address> }`. Because of that domain, a signature produced for Seaport on Polygon is useless on Ethereum mainnet or against any other contract, which is the replay protection EIP712 exists to provide. The backend recomputes it with `computeDigest(components, 137n, seaportAddress)`.

The signature is the seller's wallet's ECDSA signature over that digest, 65 bytes (`r`, `s`, `v`) serialised as a `0x` hex string, or the 64 byte EIP2098 compact form. When MetaMask shows the user a nicely formatted "Seaport wants you to sign: offer, consideration..." prompt, that is `eth_signTypedData_v4` doing EIP712 for them. On the backend, `ethers.recoverAddress(digest, signature)` runs the elliptic curve maths backwards and tells you which address must have produced the signature; if that address is the offerer, the signature is genuine. This is the entire trick that makes off chain orders trustworthy: the backend never has to trust what the frontend tells it, because it can recompute the digest from the submitted fields and check the signature against it independently. Change one wei in the price and the digest changes, and the recovered address becomes some random stranger.

The signature is also the one field that actually gives an order power. With the components and the signature, anyone in the world can call Seaport's `fulfillOrder` and buy the domain. Without the signature, the components are just a description. That is why the entity calls the `signature` column `Load-bearing - without it no order can ever fill`, and why file `05` spends time on when the public `GET /:orderHash` route is and is not allowed to return it.

A small wire format detail: the signed struct ends in `counter`, but the struct Seaport's `fulfillOrder` takes as input (`OrderParameters`) replaces `counter` with `totalOriginalConsiderationItems`. Seaport looks up the counter itself at fill time. The shared package's `toOrderParameters` does that conversion on the fill side; the backend stores the signed form (with `counter`) in `rawOrder`.

## Why fees use FEE_BPS and FEE_RECIPIENT

The marketplace takes its cut by being one of the consideration recipients. There is no escrow and no separate fee transaction; the buyer's single fill transaction sends the seller's share to the seller and the fee share to the fee wallet, atomically, or reverts entirely. Two values from AWS Secrets Manager control this, loaded by `src/components/marketplacev2/config/chain-config.loader.ts` from the secret named by `AWS_MANAGER` (region hard coded to `us-east-1`):

| Secret key | Field on `ChainConfig` | Meaning and validation |
|---|---|---|
| `FEE_BPS` | `feeBps` | The fee in basis points, where 10000 basis points is 100 percent, so `250` means 2.5 percent. `validateFeeBps` requires an integer from `1` to `10000`; zero is forbidden because a zero fee leg is something Seaport reverts on at fill time. |
| `FEE_RECIPIENT` | `feeRecipient` | The wallet that receives consideration leg one. Run through `ethers.getAddress`, so it is stored checksummed. |
| `SEAPORT_ADDRESS_POLYGON` | `seaportAddress` | The Seaport contract, used both as the EIP712 `verifyingContract` and as the operator in the approval check. |
| `USDT_ADDRESS_POLYGON` | `usdtAddress` | The only accepted payment token. |
| `DOMAIN_NFT_ADDRESS_POLYGON_UD` | `domainNftAddress` | The only accepted ERC721 contract. |
| `POL_RPC_URL` | `polygonRpcUrl` | The paid Polygon RPC endpoint (renamed from `POLYGON_RPC_URL` on 29 September 2026). Never echoed in error messages because it carries the API key. |
| `POLLER_START_BLOCK_POLYGON` | `pollerStartBlock` | Poller cursor start; falls back to `TEMPORARY_FALLBACK_POLLER_START_BLOCK` (`93_840_000`) with a warning if absent. |
| `POLLER_ENABLED` | `pollerEnabled` | `"true"` or `"false"`; also gates the listing expiry scheduler (file `05`). Falls back to `NODE_ENV === 'production'` when absent. |

The chain id itself is not configurable; `MARKETPLACEV2_CHAIN_ID = 137` lives in `config/chain-config.constants.ts` with the comment `Fixed for the demo deployment (Polygon mainnet) - not part of the AWS secret.`

Why basis points instead of a percentage float? Because money on chain is integers, and floating point is a disaster for money. USDT on Polygon has six decimals, so 100 USDT is the integer `100_000000`. The split is computed with pure integer maths, `fee = total * feeBps / 10000` with integer division truncating down, and `sellerAmount = total - fee`, so the two legs always sum to exactly the total with no rounding residue. At 250 basis points, a 100 USDT listing becomes a seller leg of `97_500000` and a fee leg of `2_500000`, the exact fixture every spec file uses. Truncating the fee rather than the seller's share means any rounding ever favours the seller by at most one minor unit.

Why does the backend recompute the split itself instead of trusting the frontend's numbers? Because the seller signs the consideration, and a malicious or buggy client could sign an order paying the fee wallet zero, or paying "the fee" to their own second wallet. Since anyone can fill any validly signed order straight on Seaport, the backend's only leverage is refusing to store and advertise orders that do not pay the marketplace correctly. That is check 5, `FEE_MISMATCH`, in file `04`. Putting the fee policy in the secret rather than in code means operations can change it without a deploy, though note that changing it only affects newly created orders: every already signed order carries its own amounts forever.

## The order table, column by column

```ts
// src/components/marketplacev2/order/entity/order.entity.ts
const bigintStringTransformer: ValueTransformer = {
    to: (value: string) => value,
    from: (value: string) => value
};

@Entity({ name: 'tbl_marketplacev2_orders' })
@Index('idx_marketplacev2_orders_status', ['status'])
@Index('idx_marketplacev2_orders_maker', ['maker'])
@Index('idx_marketplacev2_orders_token_id', ['tokenId'])
@Index('idx_marketplacev2_orders_status_created_at', ['status', 'createdAt'])
@Index('idx_marketplacev2_orders_active_listing_unique', ['tokenContract', 'tokenId'], { unique: true, where: `"status" = 'active'` })
export class OrderEntity {
    @PrimaryColumn({ type: 'varchar', length: 66 })
    public orderHash: string;
    // ...
}
```

Notice first what this entity is not. It does not extend the shared `BaseEntity` from `src/@core/common/entity/base.entity.ts`, so it has no UUID `id`, no `isDeleted` soft delete flag, no `createdDateTime` or `lastChangedDateTime`. The natural key, the order hash, is the primary key, and rows are never deleted: an order's lifecycle is expressed entirely through `status`. The table name follows the house `tbl_` convention with a `marketplacev2_` namespace.

The `bigintStringTransformer` is a deliberate no op. Postgres `bigint` columns come back from node postgres as strings already, so the transformer changes nothing today; its comment explains it exists to make that contract "explicit and immune to a driver change ever routing a value through a JS number." This matters because JavaScript numbers lose precision above `Number.MAX_SAFE_INTEGER` (about 9 times 10 to the 15), and a USDT amount in minor units or a far future timestamp should never silently round. The same thinking explains why `tokenId` is a `varchar`, not even a `bigint`: ERC721 token ids are `uint256`, and Unstoppable Domains token ids are namehashes, numbers up to 78 digits long that do not even fit in Postgres `bigint`.

| Column | TypeORM type | Null | Default | Notes |
|---|---|---|---|---|
| `orderHash` | `varchar(66)` primary key | no | none | `0x` plus 64 hex characters. The EIP712 struct hash, computed server side. Primary key uniqueness is what powers `DUPLICATE_ORDER` (check 12). |
| `status` | `enum` `OrderStatus` | no | none | `active`, `cancelled`, `filled`, `expired`, `invalid`. Indexed alone and together with `createdAt`. |
| `maker` | `varchar` | no | none | The offerer, checksummed via `getAddress` before storage. Indexed. |
| `tokenContract` | `varchar` | no | none | The real ERC721 address (checksummed), "not a provider name." Part of the partial unique index. |
| `tokenId` | `varchar` | no | none | Decimal string of the `uint256` id. Indexed alone and in the partial unique index. |
| `chainId` | `int` | no | none | Always `137` today. |
| `priceUsdt` | `bigint` (string transformer) | no | none | Total price in USDT minor units, six decimals. The comment says `sellerUsdt + feeUsdt` must equal it exactly, and that this is enforced in the service layer rather than by the schema. |
| `feeUsdt` | `bigint` (string transformer) | no | none | Taken from consideration leg one as signed, "not recomputed for storage." |
| `sellerUsdt` | `bigint` (string transformer) | no | none | Consideration leg zero. |
| `startTime` | `bigint` (string transformer) | no | none | Unix seconds, as signed. |
| `endTime` | `bigint` (string transformer) | no | none | Unix seconds, as signed. Compared against "now" by every servable read and by the expiry sweep. |
| `counter` | `varchar` | no | none | The signed Seaport counter, stored verbatim and never normalised. |
| `salt` | `varchar` | no | none | Decimal string of the 256 bit salt. |
| `conduitKey` | `varchar` | no | none | Always the zero bytes32 in practice. |
| `signature` | `text` | no | none | The seller's EIP712 signature exactly as submitted. |
| `rawOrder` | `jsonb` | no | none | The submitted `parameters` object (the signed `OrderComponents` with `counter`, all numbers as decimal strings), so a buyer can rebuild the exact fill call. |
| `domainName` | `varchar` | no | none | Denormalised from the request "for browse/search," verified against `tbl_domain_detail_bc` by check 9. |
| `tld` | `varchar` | no | none | Derived by `splitDomainNameAndTLD`, or `''` if absent. |
| `createdAt` | `timestamp` | no | none | Set explicitly to `new Date()` in `create()`; no `@CreateDateColumn`. |
| `filledAt` | `timestamp` | yes | null | "Written only by the (future) poller." |
| `fillTxHash` | `varchar` | yes | null | The fill transaction hash, written by the poller. |
| `filledBy` | `varchar` | yes | null | The buyer, "taken from OrderFulfilled.recipient," written only by the poller, and never inferred from an ERC721 `Transfer` log. |
| `invalidReason` | `enum` `OrderInvalidReason` | yes | null | Set only when `status` is `invalid`, by the poller's domain contract watcher. |
| `userId` | `varchar` | yes | null | The maker's internal user id captured at create time from the JWT. Nullable for rows created before the column was added. |

The five indexes each serve a reader. `idx_marketplacev2_orders_status` and `idx_marketplacev2_orders_status_created_at` serve browse and stats (status filtered, newest first). `idx_marketplacev2_orders_maker` serves the seller's own listing history. `idx_marketplacev2_orders_token_id` serves the poller and the "is this token listed" lookups.

The fifth, `idx_marketplacev2_orders_active_listing_unique`, is the clever one. It is a partial unique index on `(tokenContract, tokenId)` with `WHERE "status" = 'active'`. In plain words: at most one row per token may be active at a time, but any number of cancelled, filled, expired or invalid rows for the same token may coexist, so a domain can be relisted freely once its previous listing is dead. The code comment explains why a database constraint is needed on top of the service's own "is it already listed" check: "two concurrent submissions for the same token can both pass the check before either insert lands." This is the classic read then write race, and the only real fix is to let the database be the referee. File `04` shows how `create()` translates the resulting Postgres `23505` error back into a friendly `ALREADY_LISTED`.

The `userId` column carries a short war story worth reading in full, because it is a lesson in why you capture identity at write time:

```ts
// src/components/marketplacev2/order/entity/order.entity.ts
/**
 * The maker's internal userId, captured once at create() time from the
 * authenticated request - never looked up from `maker` after the fact.
 * The poller's own event handlers used to resolve userId on every fill/
 * cancel/invalidate via an exact, case-sensitive wallet-address lookup,
 * which silently failed (and skipped the listing-status/history update)
 * whenever the on-chain address's casing didn't match what's stored.
 * Nullable because rows written before this column existed have no
 * value - those still fall back to the wallet lookup.
 */
@Column({ type: 'varchar', nullable: true })
public userId: string | null;
```

Ethereum addresses are case insensitive in meaning but not in string comparison: `0xabc...` and `0xABC...` are the same wallet, yet `'0xabc' === '0xABC'` is false in JavaScript and `=` is false in SQL. The fallback lookup still exists in `OrderService.resolveOrderUserId` (`order.service.ts:523`), used by the expiry path, for legacy rows.

## Statuses and how a row moves between them

```ts
// src/components/marketplacev2/order/enum/order-status.enum.ts
export enum OrderStatus {
    ACTIVE = 'active',
    CANCELLED = 'cancelled',
    FILLED = 'filled',
    EXPIRED = 'expired',
    /** B-04 - written only by the poller's domain-contract watcher (Sprint 5), never by anything else. Always paired with a non-null `invalidReason`. */
    INVALID = 'invalid'
}
```

| Status | Who writes it | Meaning |
|---|---|---|
| `active` | `OrderService.create()` only | Signed, validated and stored. Servable to buyers only while `endTime` is still in the future (see "servable" in file `05`). |
| `cancelled` | `OrderService.cancel()` (soft, off chain) and the poller on Seaport's `OrderCancelled` event | The seller withdrew it. The backend stops advertising it, but see file `05` for why a soft cancel cannot stop a determined buyer. |
| `filled` | The poller only, on Seaport's `OrderFulfilled` | Sold. `filledAt`, `fillTxHash`, `filledBy` are populated. |
| `expired` | `OrderService.expireIfStale()`, called from `create()` and from `ListingExpiryScheduler` | `endTime` passed without a sale. |
| `invalid` | The poller's domain contract watcher only | The order can no longer fill because the world changed under it; `invalidReason` says why. |

Every transition out of `active` is written as a conditional update whose `WHERE` clause includes `status = 'active'`, so two writers racing on the same row can never both "win." The poller is also allowed to move a `cancelled`, `expired` or `invalid` row to `filled` (its `FILLABLE_FROM_STATUSES` in `poller/application/seaport-event-application.service.ts:31` lists all four), which is an honest admission that the chain is the final authority: if Seaport says it filled, it filled, whatever the database thought.

```ts
// src/components/marketplacev2/order/enum/order-invalid-reason.enum.ts
export enum OrderInvalidReason {
    /** The listed token was transferred to a wallet other than the maker. */
    OWNER_CHANGED = 'owner_changed',
    /** The maker revoked Seaport's approval for the token collection. */
    APPROVAL_REVOKED = 'approval_revoked'
}
```

These are exactly the two ways a once valid listing becomes a "ghost listing" that would revert if anyone tried to buy it: the seller no longer owns the NFT (they sold it elsewhere or transferred it), or the seller revoked Seaport's permission to move it. Both are visible as ERC721 events (`Transfer` and `ApprovalForAll`), which is why the poller, not the request path, is responsible for them.

## The reject code enum

```ts
// src/components/marketplacev2/order/enum/order-reject-code.enum.ts
export enum OrderRejectCode {
    OFFERER_NOT_CALLER = 'OFFERER_NOT_CALLER',
    BAD_SIGNATURE = 'BAD_SIGNATURE',
    BAD_OFFER_ITEM = 'BAD_OFFER_ITEM',
    BAD_CONSIDERATION = 'BAD_CONSIDERATION',
    FEE_MISMATCH = 'FEE_MISMATCH',
    BAD_ORDER_TYPE = 'BAD_ORDER_TYPE',
    BAD_TIME_WINDOW = 'BAD_TIME_WINDOW',
    STALE_COUNTER = 'STALE_COUNTER',
    NOT_OWNER = 'NOT_OWNER',
    NAME_MISMATCH = 'NAME_MISMATCH',
    NOT_APPROVED = 'NOT_APPROVED',
    ALREADY_LISTED = 'ALREADY_LISTED',
    DUPLICATE_ORDER = 'DUPLICATE_ORDER'
}
```

Thirteen codes for twelve numbered checks (check 9 produces two codes, `NOT_OWNER` and `NAME_MISMATCH`). They are the machine readable contract between backend and frontend: every rejection is a `400` whose body carries one of these in a `code` field, so the UI can switch on the code instead of parsing English. File `04` maps every one to its exact trigger. Note that this is a departure from the house convention described in the root `CLAUDE.md` ("No centralised error code registry"), and a welcome one.

## The other enums in the folder

Three more enum files live in `order/enum/`. They belong to the read side, which another file in this cluster explains in depth, but for completeness: `domain-listing-status.enum.ts` defines `DOMAIN_LISTING_STATUS_VALUES = ['LISTED', 'UNLISTED', 'SOLD']` (a domain's status in `GET /my-domains`) and `MY_DOMAINS_STATUS_FILTER_VALUES`, which adds `'ALL'` as an explicit "no filter" value for the query string. `listing-category-filter.enum.ts` defines `LISTING_CATEGORY_FILTERS = ['active', 'sold', 'cancelled', 'expired', 'expiring_soon']` for `GET /my-listings`, with a comment telling you to add the matching clause to `OrderService.CATEGORY_FILTERS` (`order.service.ts:167`) whenever you add a value. They are written as `as const` arrays rather than TypeScript enums so the same array can feed both the type and a runtime `@IsIn` validator.

## The watchlist table, briefly

The watchlist service is covered elsewhere in this cluster, but its table sits in the same `entity/` folder and is worth knowing at the column level:

```ts
// src/components/marketplacev2/order/entity/watchlist.entity.ts
@Entity({ name: 'tbl_marketplacev2_watchlist' })
@Unique('uq_marketplacev2_watchlist_user_token_contract_token_id', ['userId', 'tokenContract', 'tokenId'])
@Index('idx_marketplacev2_watchlist_user_id', ['userId'])
export class WatchlistEntity { /* ... */ }
```

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | `uuid`, `@PrimaryGeneratedColumn('uuid')` | no | Used by `DELETE /watchlist/:id`. |
| `userId` | `varchar` | no | `req.user.userId`; a preference tied to the login only, with no wallet check. Indexed. |
| `tokenContract` | `varchar` | no | Checksummed. |
| `tokenId` | `varchar` | no | Numeric string. |
| `createdAt` | `timestamp` | no | Set by the service. |

The unique constraint on `(userId, tokenContract, tokenId)` makes "add to watchlist" idempotent at the database level. The header comment explains why this is a brand new table instead of v1's `tbl_watchlist`: v1 keyed watchlist rows on `DomainListing.id`, a per listing UUID that changes every time a domain is relisted, so a watched domain silently fell off your watchlist when its seller relisted it. Keying on `(tokenContract, tokenId)`, the domain's stable on chain identity, fixes that by construction.

## The shared order builder package

Signing happens in the browser and verification happens on the server, and the two must agree on every byte. If the frontend's idea of the EIP712 types differs from the backend's by so much as the order of two fields, every signature the frontend produces will recover to a random address on the backend and every listing will fail with `BAD_SIGNATURE`. The team's answer is a shared package, `@endlessdomains/order-builder`, pinned in `package.json` to an exact git commit:

```ts
// package.json (dependencies)
"@endlessdomains/order-builder": "git+https://github.com/Endless-Domains/ed-shared-package.git#855e344aa2be0ebe6287a57de507a71d3be5566e",
```

The lockfile records it as version `0.1.0`, `UNLICENSED`, with a single dependency, `ethers ^6.13.4`. Pinning to a commit hash rather than a branch is the right call: a push to the shared repo can never change hashing behaviour under a running backend without someone deliberately bumping the hash. (The package was not installed in the working copy reviewed for these notes, so its surface is reconstructed from the backend's imports, the specs that exercise it, and the frontend's field for field mirror described below.)

What the backend borrows from it, by import site:

| Export | Where the backend uses it | What it does |
|---|---|---|
| `ItemType` | `order.service.ts`, `offer-item.dto.ts`, `consideration-item.dto.ts` | Seaport item types: `NATIVE 0`, `ERC20 1`, `ERC721 2`, `ERC1155 3`. The DTOs validate with `@IsIn(Object.values(ItemType))`. |
| `OrderType` | `order.service.ts`, `order-parameters.dto.ts` | `FULL_OPEN 0`, `PARTIAL_OPEN 1`, `FULL_RESTRICTED 2`. |
| `ZERO_ADDRESS`, `ZERO_BYTES32`, `NO_CONDUIT_KEY` | `order.service.ts` check 6 | The required values for `zone`, `zoneHash`, `conduitKey`. |
| `OrderComponents` | `order.service.ts` | The TypeScript type of the signed struct with `bigint` amounts. |
| `computeOrderHash(components)` | `validateLocal()` return value | EIP712 `hashStruct`, the primary key. |
| `computeDigest(components, chainId, seaport)` | check 2 | The full EIP712 digest that is signed. |
| `computeSplit(total, feeBps)` | check 5 | Integer fee split, fee truncated down, seller takes the remainder; throws `OrderError` on bad input. |
| `OrderError` | check 5 | The package's own error class, caught and converted to `FEE_MISMATCH`. |
| `buildOrderComponents(params)` | only in `order-builder-wiring.smoke.spec.ts` | The frontend's entry point: builds a complete, policy compliant order from a seller, token, price and fee. |

The package also exports things the backend does not import but the frontend uses: `toOrderParameters` (signed struct to fill struct), the EIP712 type table, a `seaportDomain(verifyingContract, chainId)` helper, and minimal ABIs.

There is an important wrinkle on the frontend side. The package needs ethers v6, but the marketplace frontend is on ethers 5.7.2, so the frontend carries `src/lib/seaport-order-builder-v5.ts`, a hand written ethers v5 reimplementation whose header says it mirrors the real package's constants and structure field for field. Its own comment calls it "a deliberate, temporary duplication for a throwaway test page, not a second production implementation," but in the current frontend it is imported by the real listing flow (`src/design-system/composites/my-domains/listing-flow/useListingFlowActions.ts`) and buying flow (`useBuyFlowActions.ts`). So in practice the frontend does not use the shared package at all; it uses a copy. That is precisely the drift risk the package was created to eliminate, and the only thing guarding against it is the parity tests described next.

## Parity, and the two tests that defend it

"Parity" in this codebase means two related promises. First, that an order built and signed by the frontend is byte for byte the order the backend validates, so hashes and signatures agree across the network boundary. Second, that the backend's twelve checks reject exactly the twelve kinds of bad order the plan specified, no more and no fewer. Two spec files defend these promises.

`order/order-builder-wiring.smoke.spec.ts` is a single test that pins the hashing algorithm to a known answer:

```ts
// src/components/marketplacev2/order/order-builder-wiring.smoke.spec.ts
const components = buildOrderComponents({
    seller: '0x1111111111111111111111111111111111111111',
    counter: 0n,
    now: 1700000000n,
    salt: 123456789n,
    domainNft: '0x2222222222222222222222222222222222222222',
    tokenId: 42n,
    usdt: '0x3333333333333333333333333333333333333333',
    feeRecipient: '0x4444444444444444444444444444444444444444',
    totalMinorUnits: 100_000000n,
    feeBps: 250n
});

expect(computeOrderHash(components)).toBe(
    '0xec2b15363a346c75217b5d1948432717fe8ea4bc15a7be8993f466063d64cafe'
);
```

Its comment says it proves two things: that the git dependency resolves and is callable under this repo's ts jest setup, and that `computeOrderHash` still produces the exact hash that was independently verified against the original `seaport-polygon-spike` source during extraction. If a future bump of the pinned commit ever changes field order, type names, or the backdating rule, this hash changes and the test fails immediately. It is a golden master test, and it is the cheapest possible insurance on the most expensive possible bug. What it does not do is run the frontend's v5 copy against the same fixture; a matching golden test in the frontend repo would close that loop, and I would recommend adding one.

`order/order.parity.spec.ts` is, in its own words, the `B-03 Sprint 5` full parity suite covering all twelve rejection cases plus the happy path. Every case goes through `create()` end to end with a real randomly generated `ethers.Wallet` signing a real digest, and with the chain contracts and repositories replaced by Jest mocks. Its cases are: 0, the happy path inserts exactly once and returns a lowercase 64 hex order hash; 1, `OFFERER_NOT_CALLER` when a different wallet calls; 2, `BAD_SIGNATURE` after flipping one hex digit in the signature's `r` component; 3, `BAD_OFFER_ITEM` when the offer token is the USDT address; 4, `BAD_CONSIDERATION` with only the seller leg present; 5, `FEE_MISMATCH` with a 90/10 split instead of 97.5/2.5; 6, `BAD_ORDER_TYPE` with `PARTIAL_OPEN`; 7, `BAD_TIME_WINDOW` for an already expired window; 8, `STALE_COUNTER` with the chain returning counter `5`; 9a, `NOT_OWNER` with `ownerOf` returning someone else; 9b, `NAME_MISMATCH` with the database record naming `someone-else.eth`; 10, `NOT_APPROVED`; 11, `ALREADY_LISTED` when `findOne` returns an existing order; 12, `DUPLICATE_ORDER` when `insert` rejects with `{ code: '23505' }`; plus a separate case asserting a `503` `CHAIN_UNAVAILABLE` (not a `400`) when `getCounter` rejects with "could not detect network."

The suite's header is candid about what it does not cover: confirming that an accepted order can actually be filled on Polygon "needs a funded wallet that actually owns an approved domain NFT on mainnet and a real, irreversible transaction," which was left to a human. That is the right boundary for a unit suite, but it does mean no automated test proves the stored `rawOrder` plus `signature` actually fills on Seaport.

## The entity spec

`order/entity/order.entity.spec.ts` has four tests that guard the schema decisions above using TypeORM's `getMetadataArgsStorage()`, which lets a test inspect decorator metadata without a database. `B-04 TC3.1` asserts `OrderStatus` has exactly the five values `active`, `cancelled`, `expired`, `filled`, `invalid`, so nobody renames or drops one. The second test asserts `invalidReason` and `filledBy` are both nullable, that `invalidReason`'s enum is `OrderInvalidReason`, that the enum has exactly `owner_changed` and `approval_revoked`, and that `filledBy` is a `varchar`. The third round trips a `tokenId` of `'99999999999999999999'` (bigger than `Number.MAX_SAFE_INTEGER`) through `JSON.stringify` and back and checks it is still the identical string. The fourth finds every `bigint` column, asserts the set is exactly `endTime`, `feeUsdt`, `priceUsdt`, `sellerUsdt`, `startTime`, and runs each one's transformer in both directions to prove it returns the same huge string unchanged.

## Problems and inconsistencies worth knowing

The entity's header comment (`order.entity.ts:15` to `19`) still says the table name "is a placeholder pending confirmation before Sprint 3's migration" and is "not synced to any database in this sprint." That is no longer true: the module is wired into `Marketplacev2Module`, which `app.module.ts:101` imports, and the routes are live. There is also no checked in migration for any `tbl_marketplacev2_*` table; with `synchronize: false` (`src/@core/config/type-orm-config.service.ts:73`), the schema, including the partial unique index that the whole `ALREADY_LISTED` race protection depends on, only exists because `run/deploy.sh` runs `typeorm:migration:generate -n "update"` followed by `migration:run` on every deploy. If that autogenerated migration ever failed to emit the partial index's `WHERE` clause correctly, the race protection would silently vanish, and nothing in the test suite would notice, because every spec mocks the repository.

`createdAt` and `filledAt` are `timestamp` (without time zone) while the rest of the codebase's `BaseEntity` uses `timestamptz`. node postgres serialises a JavaScript `Date` in the server process's local time zone, so these columns are only correct as long as every server, and every developer laptop pointed at a shared database, runs in UTC. A developer in India writing to the UAT database would stamp rows five and a half hours off.

`package.json` declares the order builder dependency as `git+https://...` but `package-lock.json:3815` resolved it as `git+ssh://git@github.com/...`. An `npm ci` on a build machine without an SSH key for that private repo can fail even though the HTTPS URL in `package.json` looks fine. Worth checking against the CodeBuild environment.

The frontend's production listing and buying flows depend on an ethers v5 copy of the package that describes itself as throwaway. If anyone changes the shared package (say, a new field, or a different backdate), the smoke test will catch it on the backend, but the frontend copy will drift silently and every listing will start failing with `BAD_SIGNATURE`.

## Frontend note

If you are coming from the frontend, the single most useful mental shift here is that the signature is the listing. In v1 the listing lived in a smart contract and the database was a cache of it. In v2 the listing lives in this Postgres table, and the chain only finds out about it at the moment someone buys. Everything that follows in files `04` and `05` is the backend trying hard to make sure the rows it stores and advertises are ones that would actually succeed if a buyer handed them to Seaport right now, while being honest that the chain, not the database, has the final word.
