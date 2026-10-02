# 08. The On Chain Event Poller, Architecture

## What this file covers, and the one sentence version

Marketplace v2 lets a seller sign a Seaport order off chain, stores that signed order in `tbl_marketplacev2_orders`, and then lets anybody fill it on Polygon by calling the Seaport contract directly. The backend is never in the middle of that fill. Nobody sends the backend a transaction hash, nobody calls a webhook, nobody hits an endpoint saying "I just bought this". The only way the backend can ever learn that a listing sold, was cancelled on chain, or quietly became unfillable because the seller moved the domain away, is to go and read the blockchain itself. The folder `src/components/marketplacev2/poller/` is the piece that does that reading, every fifteen seconds, block by block, and the one sentence version of it is this: a single row cursor remembers the last fully processed Polygon block, every tick asks the RPC node for all Seaport and domain NFT logs in the next slice of confirmed blocks, hands each log to an application service that applies it to the database with conditional, idempotent writes, and only then moves the cursor forward.

This note is the architecture half. It explains why polling logs was chosen over websockets and contract listeners, what a log, a topic, a block range and a confirmation depth actually are, how the Nest module is wired, how the poller is switched on or off per environment, the full tick loop line by line, the cursor table column by column, the retry and backoff utilities, the error classification, the concurrency guard, the topic assertion that runs at boot, the two controller endpoints and their guards, and finally how this compares to the older `listener` module (file `05/09`) and the v1 transaction cron (file `03/04`). The companion note, `09-applying-seaport-and-domain-events.md`, goes event by event through what each decoded log actually does to the database, and walks every spec test.

Everything here was read against the `uat` branch, which landed this whole folder in seven commits by the same author between 21 September and 1 October 2026: `546e26e9` (the initial poller system, 2,449 lines in one commit), `b8c0f074` (listing status and history writes, block timestamps), `de4d906e` (the new transactions table), `79a63770` (constant reverts and a logging tweak during an RPC incident), `1461bda2` (the `POLLER_ENABLED` flag), `96f221ec` (stale replay guards and the `userId` column) and `bdd8e80a` (deterministic versus transient error classification, and the admin guard on `_health`). Reading those diffs in order is genuinely instructive, because several of the most important lines in the current code exist only because something went wrong in UAT, and the comments say so.

## A stack note before anything else

The project `CLAUDE.md` still says NestJS 8, TypeORM 0.2 and ethers 5. That is out of date on this branch. `package.json` on `uat` pins `@nestjs/core` `^10.4.22`, `typeorm` `^0.3.17`, and, new in this batch (commit `71fd2fec`, "Ether versoin 6"), `ethers` `^6.13.4`. Every ethers call in the poller uses the version 6 API: `Interface`, `JsonRpcProvider`, `getAddress` and `Log` imported straight from the package root, native JavaScript `bigint` for every decoded `uint256`, `fragment.topicHash` for topic derivation, and `receipt.gasPrice` (which in version 6 is the effective gas price; version 5 called it `effectiveGasPrice`). Keep that in mind when you read older notes in this repo that show `ethers.utils` or `BigNumber`.

## Why poll logs at all, a primer for a frontend developer

### What an event log is

When a smart contract function runs, it can `emit` an event. An event is not stored in the contract's state, and the contract itself can never read it back. It is written into the transaction receipt as a log entry, and every Ethereum style node indexes those log entries so that outside programs can ask "show me every log this contract emitted between block X and block Y". Logs are the blockchain's equivalent of an append only audit trail, and they are by far the cheapest and most reliable way for an off chain system to learn what happened on chain.

Each log has three parts that matter here. The `address` is the contract that emitted it. The `topics` are up to four 32 byte values that the node indexes and lets you filter on. The `data` is an arbitrary length blob holding every parameter that was not marked `indexed`, ABI encoded. On top of those, the node attaches metadata such as `blockNumber`, `transactionHash` and the log's index inside the block.

### What a topic is, and why the first topic is special

`topics[0]` is, for every normal (non anonymous) event, the keccak256 hash of the event's canonical signature, for example `keccak256("Transfer(address,address,uint256)")`, which is the famous `0xddf252ad...` value. That hash is how you know which event a log is. `topics[1]` to `topics[3]` hold the values of up to three parameters declared `indexed`, which is why you can ask a node "give me every `Transfer` whose `to` is my wallet" but you cannot ask it "give me every `ApprovalForAll` where `approved` is false", because `approved` is not indexed and lives in `data`. The poller's ABI file says this outright for `ApprovalForAll`, and the domain application service always decodes `approved` from `data` rather than filtering on it.

The critical consequence for this codebase is that the signature hash is computed from the exact parameter types, including the exact shape of every tuple. Get one tuple field wrong in your ABI string and you derive a different `topics[0]`, your filter matches nothing, and your poller sits there forever, perfectly healthy, finding no sales. That failure is silent, which is why this module has a boot time assertion against known good hashes, covered below.

### What a block range is

`eth_getLogs` takes a `fromBlock` and a `toBlock`. Nodes and RPC providers cap how much you can ask for in one call, either by block span (some providers allow 2,000 or 10,000 blocks) or by result count (Infura style "more than 10,000 results" errors). A poller therefore walks the chain in slices. This module calls the slice size `MAX_RANGE` and sets it to 500 blocks.

### What a confirmation depth is, and what a re org is

Blockchains occasionally reorganise their most recent blocks. Two validators produce competing blocks at the same height, the network briefly disagrees, and eventually one branch wins and the other branch's blocks are discarded. Any log in a discarded block simply stops existing. That is a re org. A naive poller that reads the very latest block can record a sale that, ten seconds later, never happened.

There are two honest ways to deal with this. Either you implement re org handling (remember block hashes, detect when a block you already processed has a different hash now, and roll back everything you derived from it), or you stay far enough behind the head of the chain that re orgs deeper than that distance are vanishingly rare, and accept the residual risk. The second approach is called a confirmation depth. This module takes the second approach with `CONFIRMATIONS = 5`, and its own comment is admirably blunt about it.

```ts
// src/components/marketplacev2/poller/constants/poller.constants.ts
/** Stay this far behind the chain head. The only substitute for reorg handling, which this poller deliberately does not have. */
export const CONFIRMATIONS = 5;
```

### Why polling instead of websockets or `contract.on(...)`

An ethers `Contract` instance can call `.on('Transfer', handler)` and a websocket provider can push new logs to you as they arrive. That feels more "real time" and is exactly what file `05/09` warned you not to assume the old `listener` module was doing. For a backend that must never lose a sale, subscriptions have real problems. A websocket connection drops silently and you miss everything emitted while it was down, with no built in way to know what you missed. A subscription pushes logs from the very latest block, so you are exposed to re orgs unless you add your own delay. Restarting the process (a deploy, a crash, a PM2 reload) loses whatever was in flight, and on restart you have no record of where you were. And free or cheap RPC tiers often do not offer websocket endpoints at all.

Polling `eth_getLogs` against a durable cursor fixes all of these at once. The cursor is in Postgres, so a restart resumes exactly where it stopped. A failed request just means the cursor does not move and the same range is retried. Staying `CONFIRMATIONS` blocks behind the head is a one line subtraction. And the whole thing is plain HTTP JSON RPC, which every provider supports. The price you pay is latency: with a fifteen second tick and a five block buffer of roughly two second Polygon blocks, a sale typically shows up about twenty to thirty seconds after it lands, which the constants file quotes from the original brief as "fast enough that a demo feels live, slow enough not to hammer the RPC".

The code itself is honest that this is a stopgap. The tick loop class is described as "the throwaway half", meant to be replaced by a real indexer later behind the same `ChainEventSource` interface, while the two application services are "the reusable half" that the future indexer will call too.

## The files, and how they fit together

The folder splits into four layers.

| Layer | Files | Role |
|---|---|---|
| ABI and boot safety | `abi/poller-events.abi.ts`, `abi/assert-event-topics.ts`, `abi/poller-topic-assertion.error.ts` | Human readable event fragments for Seaport and ERC721, derivation of their topic hashes, and a hard failure at boot if the derived hashes do not match known good values |
| State change ("reusable half") | `application/seaport-event-application.service.ts`, `application/domain-event-application.service.ts`, `application/decodable-log.interface.ts` | Given one already fetched log, decode it and apply it to `tbl_marketplacev2_orders` and the related tables, idempotently |
| Log fetching ("throwaway half") | `tick/chain-event-source-poller.service.ts`, `constants/poller.constants.ts`, `entity/poller-cursor.entity.ts` | The setInterval loop, range sizing, `getLogs`, cursor read and write, error classification, health status |
| Wiring and HTTP | `poller.module.ts`, `poller.service.ts`, `poller.controller.ts`, `interface/*.ts` | Nest module, cursor seeding and topic assertion at boot, two health endpoints |

`DecodableLog` deserves a sentence now because it is the seam between the halves. It is a deliberately minimal shape, `topics`, `data`, `transactionHash`, `blockNumber`, and an optional `blockTimestamp`, rather than ethers' own `Log` type, so the application services can be unit tested with hand built fixtures and do not depend on the fetching loop at all. An ethers `Log` structurally satisfies it.

```ts
// src/components/marketplacev2/poller/application/decodable-log.interface.ts
export interface DecodableLog {
    readonly topics: ReadonlyArray<string>;
    readonly data: string;
    readonly transactionHash: string;
    readonly blockNumber: number;
    readonly blockTimestamp?: number;
}
```

## Module wiring

```ts
// src/components/marketplacev2/poller/poller.module.ts
@Module({
    imports: [LoggerModule, TypeOrmModule.forFeature([PollerCursorEntity, OrderEntity]), ChainConfigModule, ListingStatusModule, TransactionModule, WalletAddressModule],
    controllers: [PollerController],
    providers: [
        { provide: 'PollerServiceInterface', useClass: PollerService },
        SeaportEventApplicationService,
        DomainEventApplicationService,
        { provide: 'ChainEventSource', useClass: ChainEventSourcePoller }
    ],
    exports: [ /* the same four providers */ ]
})
export class PollerModule {}
```

Read the imports as a list of everything the poller writes to or reads from. `TypeOrmModule.forFeature([PollerCursorEntity, OrderEntity])` gives it repositories for its own cursor table and, notably, for the orders table owned by `OrderModule`. The module comment explains why: the application services write `tbl_marketplacev2_orders` directly, and importing `OrderModule` would create a sibling dependency the team wanted to avoid. `ListingStatusModule` brings `ListingStatusService` and `ListingHistoryService` (the "my domains" status cache and the append only listing timeline), `TransactionModule` brings `TransactionService` (the financial settlement table), `WalletAddressModule` brings the `WalletRepoInterface` used as a fallback to map a maker wallet to an internal `userId`, and `ChainConfigModule` provides the `CHAIN_CONFIG` token.

`PollerModule` is imported by `Marketplacev2Module` (`src/components/marketplacev2/marketplacev2.module.ts`), which also re exports it. The exports block repeats the provider objects rather than listing tokens. Nest resolves exported custom providers by their `provide` token, so this does not create a second instance, but it reads as if it might, and listing the tokens (`'PollerServiceInterface'`, `'ChainEventSource'`) would be clearer.

The `'ChainEventSource'` token is the forward looking part of the design. `ChainEventSourcePoller` is registered under an interface token, so a future indexer can be swapped in at that one line without the controller or anything else changing.

```ts
// src/components/marketplacev2/poller/interface/chain-event-source.interface.ts
export interface ChainEventSource {
    /** Begin consuming events. Idempotent - safe to call twice. */
    start(): Promise<void>;
    stop(): Promise<void>;
    /** How far behind the chain head we are, in blocks. For health checks. */
    getLag(): Promise<number>;
    getStatus(): Promise<PollerStatus>;
}
```

Nobody has to call `start()` from outside. `ChainEventSourcePoller` implements `OnModuleInit` and `OnModuleDestroy` itself, so Nest starts the loop when the application boots and would stop it on shutdown. One caveat on that second half: `src/main.ts` does not call `app.enableShutdownHooks()`, so on a real `SIGTERM` from PM2 or ECS, `onModuleDestroy` is never invoked and the interval simply dies with the process. Because the cursor only advances after a range is fully applied, and every write is a short transaction, this is harmless for correctness, but it means `stop()` is effectively only exercised by tests.

## Chain config, and where every value comes from

All chain values come from the same AWS Secrets Manager secret every other module reads (`AWS_MANAGER`), loaded once at boot by `ChainConfigModule`'s async factory.

```ts
// src/components/marketplacev2/config/chain.config.ts
export interface RawChainConfigSecret {
    POL_RPC_URL?: string;
    SEAPORT_ADDRESS_POLYGON?: string;
    USDT_ADDRESS_POLYGON?: string;
    DOMAIN_NFT_ADDRESS_POLYGON_UD?: string;
    FEE_RECIPIENT?: string;
    FEE_BPS?: string;
    POLLER_START_BLOCK_POLYGON?: string;
    /** "true" or "false". */
    POLLER_ENABLED?: string;
}
```

| Secret key | Becomes | Used by the poller for |
|---|---|---|
| `POL_RPC_URL` | `polygonRpcUrl` | Every `JsonRpcProvider` the poller and its services build. Renamed from `POLYGON_RPC_URL` on 29 September 2026 when the team moved to a paid endpoint because the free tier "couldn't serve eth_getLogs far enough behind the chain head for the poller to ever catch up" |
| `SEAPORT_ADDRESS_POLYGON` | `seaportAddress` | The `address` filter on the Seaport `getLogs` call, and the operator check for `ApprovalForAll` |
| `DOMAIN_NFT_ADDRESS_POLYGON_UD` | `domainNftAddress` | The `address` filter on the domain `getLogs` call, and the `tokenContract` match for `Transfer` |
| `FEE_RECIPIENT` | `feeRecipient` | Verifying the fee leg of an `OrderFulfilled` |
| `POLLER_START_BLOCK_POLYGON` | `pollerStartBlock` | Seeding the cursor the first time the row is created |
| `POLLER_ENABLED` | `pollerEnabled` | Whether this process starts the tick loop at all |
| (none, hardcoded) | `MARKETPLACEV2_CHAIN_ID = 137` | Passed to every `JsonRpcProvider` and written to transaction rows |

Every address is passed through `validateChecksumAddress`, which returns `ethers.getAddress(value)`, so all configured addresses are normalised to EIP 55 checksum casing. That matters because the orders table stores `maker` and `tokenContract` checksummed too, and several poller queries compare them with plain equality.

### The start block fallback

```ts
// src/components/marketplacev2/config/chain-config.loader.ts
export const TEMPORARY_FALLBACK_POLLER_START_BLOCK = 93_840_000;
...
function resolvePollerStartBlock(value: string | undefined): number {
    if (value === undefined || value === null || value === '') {
        console.warn(`[chain-config] POLLER_START_BLOCK_POLYGON is missing from the AWS secret - falling back to ${TEMPORARY_FALLBACK_POLLER_START_BLOCK} temporarily. ...`);
        return TEMPORARY_FALLBACK_POLLER_START_BLOCK;
    }
    return validateStartBlock('POLLER_START_BLOCK_POLYGON', value);
}
```

A present but malformed value still throws, and zero is rejected by `validateStartBlock` so the first run never tries to scan from genesis. An absent value silently falls back to a hardcoded block. That is convenient for getting stage to boot, and risky in one specific way covered in the risk table: if the real first marketplace order was signed before block 93,840,000, any sale or invalidation before that block is never seen by a poller that booted on the fallback.

### Enabled or disabled per environment

This flag exists because of a real incident, and the loader comment tells the story.

```ts
// src/components/marketplacev2/config/chain-config.loader.ts
/**
 * Single-instance guard (B-04 brief, "single instance only"). On 2026-09-29
 * a developer's local backend was polling against the shared UAT database
 * alongside the UAT server, and the two kept overwriting the one cursor row
 * - the cursor jumped backward and the lag never dropped. Each environment's
 * own secret now says whether it polls. When the key is absent, falls back
 * to polling only under NODE_ENV=production (which PM2 sets on the servers),
 * so a local boot never polls by default. ...
 */
function resolvePollerEnabled(value: string | undefined): boolean {
    if (value === undefined || value === null || value === '') {
        const fallback = process.env.NODE_ENV?.trim() === 'production';
        console.warn(`[chain-config] POLLER_ENABLED is missing from the AWS secret - falling back to ${fallback} ...`);
        return fallback;
    }
    const normalized = String(value).trim().toLowerCase();
    if (normalized === 'true') return true;
    if (normalized === 'false') return false;
    throw new ChainConfigError(`POLLER_ENABLED must be "true" or "false", got "${value}"`);
}
```

So the decision tree is: explicit `"true"` or `"false"` in the secret wins (case and whitespace insensitive); anything else non empty fails boot; absent means "poll only if `NODE_ENV` is `production`". The `.trim()` is there because the Windows `start:dev` script (`set NODE_ENV=dev && ...`) leaves a trailing space in the variable. `pm2.config.js` sets `NODE_ENV: 'production'` for the server process.

The flag is read in exactly two places that start timers: `ChainEventSourcePoller.onModuleInit` and, outside this folder, `ListingExpiryScheduler.onModuleInit` in `order/listing-expiry.scheduler.ts`, which deliberately shares the flag because an expiry sweep running somewhere the poller is not "can mark a listing expired before the poller has seen a fill that landed in time".

```ts
// src/components/marketplacev2/poller/tick/chain-event-source-poller.service.ts
async onModuleInit(): Promise<void> {
    if (!this.chainConfig.pollerEnabled) {
        this.customLoggerService.log('poller disabled - POLLER_ENABLED is false for this environment (see the AWS secret). Not starting.');
        return;
    }
    await this.start();
}
```

Two things are worth noticing about what the flag does not cover. `PollerService.onModuleInit` still runs the topic assertion and still seeds the cursor row on a disabled instance, which is harmless because the seed is `ON CONFLICT DO NOTHING`. And the health endpoint still answers on a disabled instance, reporting `healthy: false` forever because `lastTickAt` stays `null`. That is correct behaviour, but someone looking at a UAT dashboard backed by a disabled instance could misread it as an outage.

The flag is a per environment switch, not a per process lock. It cannot prevent two processes of the same, enabled environment from polling at once. That gap is the biggest operational risk in this module and gets its own section further down.

## Boot: topic assertion and cursor seeding

`PollerService` is small, and its whole job happens in `onModuleInit`.

```ts
// src/components/marketplacev2/poller/poller.service.ts
async onModuleInit(): Promise<void> {
    assertEventTopics((message) => this.customLoggerService.debug(message));

    const seedValue = String(this.chainConfig.pollerStartBlock - 1);
    await this.cursorRepository.manager.query(`INSERT INTO "tbl_marketplacev2_poller_cursor" ("id", "lastProcessedBlock", "updatedAt") VALUES ($1, $2, NOW()) ON CONFLICT ("id") DO NOTHING`, [SINGLETON_CURSOR_ID, seedValue]);
    this.customLoggerService.log(`poller cursor ensured (pollerStartBlock=${this.chainConfig.pollerStartBlock})`);
}
```

### The topic assertion

The ABI file holds five human readable event fragments. The three Seaport ones were, per the file's header, copied verbatim from a spike project (`Shared-package/seaport-polygon-spike/src/abis.ts`) that had verified them against the deployed Seaport 1.6 bytecode on Polygon; the two ERC721 ones are standard.

```ts
// src/components/marketplacev2/poller/abi/poller-events.abi.ts
const SPENT_ITEM = '(uint8 itemType,address token,uint256 identifier,uint256 amount)';
const RECEIVED_ITEM = '(uint8 itemType,address token,uint256 identifier,uint256 amount,address recipient)';

export const SEAPORT_EVENTS_ABI = [
    `event OrderFulfilled(bytes32 orderHash, address indexed offerer, address indexed zone, address recipient, ${SPENT_ITEM}[] offer, ${RECEIVED_ITEM}[] consideration)`,
    'event CounterIncremented(uint256 newCounter, address indexed offerer)',
    'event OrderCancelled(bytes32 orderHash, address indexed offerer, address indexed zone)'
] as const;

export const ERC721_EVENTS_ABI = ['event Transfer(address indexed from, address indexed to, uint256 indexed tokenId)', 'event ApprovalForAll(address indexed owner, address indexed operator, bool approved)'] as const;
```

The comment above `SEAPORT_EVENTS_ABI` names the trap this guards against, which the team calls "Trap 1". Seaport has two families of item structs. The ones you sign (`OfferItem`, `ConsiderationItem`) have five and six fields with `startAmount` and `endAmount`. The ones the contract emits after resolving amounts (`SpentItem`, `ReceivedItem`) have four and five fields with a single `amount`. If someone "fixes" the ABI by pasting the signed struct shapes, `OrderFulfilled`'s topic hash silently changes.

`assert-event-topics.ts` turns that silent failure into a loud one.

```ts
// src/components/marketplacev2/poller/abi/assert-event-topics.ts
const KNOWN_GOOD_TOPICS = {
    OrderFulfilled: '0x9d9af8e38d66c62e2c12f0225249fd9d721c54b83f48d9352c97c6cacdcb6f31',
    Transfer: '0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef',
    ApprovalForAll: '0x17307eab39ab6107e8899845ad3d59bd9653f200f220920489ca2b5937696c31'
};

function deriveTopic(iface: Interface, eventName: string): string {
    const fragment = iface.getEvent(eventName);
    if (!fragment) {
        throw new PollerTopicAssertionError(`Event "${eventName}" is not present in the poller's ABI, but is required.`);
    }
    return fragment.topicHash;
}

export function deriveAndAssertTopics(seaportAbi: readonly string[], erc721Abi: readonly string[], onDebug?: (message: string) => void): EventTopics {
    const seaportInterface = new Interface(seaportAbi as string[]);
    const erc721Interface = new Interface(erc721Abi as string[]);
    const topics: EventTopics = {
        orderFulfilled: deriveTopic(seaportInterface, 'OrderFulfilled'),
        orderCancelled: deriveTopic(seaportInterface, 'OrderCancelled'),
        counterIncremented: deriveTopic(seaportInterface, 'CounterIncremented'),
        transfer: deriveTopic(erc721Interface, 'Transfer'),
        approvalForAll: deriveTopic(erc721Interface, 'ApprovalForAll')
    };
    // ... compare three of them against KNOWN_GOOD_TOPICS, case insensitively, throw on mismatch
    onDebug?.(`poller event topics asserted - ...`);
    return topics;
}
```

In ethers version 6, `new Interface([...human readable strings])` parses the fragments, `getEvent(name)` returns an `EventFragment` or `null`, and `fragment.topicHash` is the keccak256 of the canonical signature. The function takes the ABI as parameters purely so tests can pass a corrupted copy; `assertEventTopics()` is the real entry point bound to the real ABI.

Only three of the five topics are asserted. `OrderCancelled` and `CounterIncremented` had no known good constant in the brief, so they are derived and logged but not checked, and the spec `TC2.6` even asserts that a mutated `OrderCancelled` does not throw. That is a documented, intentional gap, and it is worth closing: if `OrderCancelled`'s fragment were ever mistyped, the poller would silently stop seeing on chain cancels. The known good value is easy to compute once and pin.

`PollerTopicAssertionError` is a plain `Error` subclass with its `name` set, so a stack trace in the boot log names it clearly. Because it is thrown from `onModuleInit`, a mismatch fails the boot of the entire API, not just the poller. That is the intended "fail loudly" behaviour, but it is worth being clear eyed that an ABI typo in this one folder takes down login, checkout and every v1 feature with it. A softer alternative would be to refuse to start the poller and flip health to unhealthy, but the team chose the hard failure on purpose so that nobody can deploy a poller that quietly finds nothing.

The assertion runs a second time inside `ChainEventSourcePoller`'s constructor, because that class needs the derived topics to build its `getLogs` filters and the comment calls it "a second line of defense against module init ordering". It is pure and cheap, so running it twice costs nothing. There is one latent hazard in both call sites, explained in the logging section: the `onDebug` callback calls `CustomLoggerService.debug`, and that logger initialises asynchronously.

### Seeding the cursor

The seed value is `pollerStartBlock - 1`, so that the first tick's `from = lastProcessedBlock + 1` lands exactly on the start block. The seed is a raw `INSERT ... ON CONFLICT ("id") DO NOTHING`, which is idempotent and race free even if two processes boot simultaneously, and which never touches an existing row. That last property has an operational consequence: changing `POLLER_START_BLOCK_POLYGON` in the secret after the first boot has no effect at all. To rewind or skip ahead you must update the row by hand.

The seed lives here rather than in a migration because the value is environment specific. The table itself has no hand written migration in `src/`; TypeORM config has `synchronize: false`, so the table is created by the deploy pipeline's generated migration step described in the project `CLAUDE.md`.

## The cursor table, column by column

```ts
// src/components/marketplacev2/poller/entity/poller-cursor.entity.ts
@Entity({ name: 'tbl_marketplacev2_poller_cursor' })
export class PollerCursorEntity {
    @PrimaryColumn({ type: 'smallint' })
    public id: number;

    @Column({ type: 'bigint', transformer: bigintStringTransformer })
    public lastProcessedBlock: string;

    @Column({ type: 'timestamp' })
    public updatedAt: Date;
}
```

| Column | Type | Meaning | Who writes it |
|---|---|---|---|
| `id` | `smallint` primary key | Always `1` (`SINGLETON_CURSOR_ID`). The table has exactly one row | Seed in `PollerService.onModuleInit` |
| `lastProcessedBlock` | `bigint`, string in JS | The highest block whose logs have been fully applied. The next tick starts at this plus one | Seed (start block minus one), then `advanceCursor` after every successful range |
| `updatedAt` | `timestamp` (no time zone) | When the cursor last moved. Seeded with the database's `NOW()`, later written with the application's `new Date()` | Same two places |

Notice what is not there. It does not extend the shared `BaseEntity`, so there is no UUID, no `isDeleted`, no `createdDateTime`. There is no `chainId` column, so a second chain means a schema change. There is no block hash, so there is nothing to detect a re org with. There is no owner or lease column, so there is nothing that identifies which process last advanced it, and nothing that could stop a second process from advancing it too. And `lastProcessedBlock` is written with an unconditional `UPDATE ... WHERE id = 1`, not a compare and swap on the previous value.

The `bigint` with a pass through transformer is a consistency choice shared with `OrderEntity`. Postgres `bigint` arrives from `node-postgres` as a string, and the transformer keeps it a string. The tick loop converts it with `Number(row.lastProcessedBlock)`, which is safe since Polygon block numbers are around 10^8, far below `Number.MAX_SAFE_INTEGER`.

## The tick loop in full

### Constants

```ts
// src/components/marketplacev2/poller/constants/poller.constants.ts
export const TICK_INTERVAL_MS = 15_000;
export const CONFIRMATIONS = 5;
export const MAX_RANGE = 500;
export const SINGLETON_CURSOR_ID = 1;
export const UNHEALTHY_LAG_BLOCKS = 300;
export const STALE_TICK_MS = TICK_INTERVAL_MS * 3;
```

The comments on `TICK_INTERVAL_MS` and `MAX_RANGE` record a small piece of incident history from commit `79a63770`. On 29 September 2026, during a free tier RPC incident, someone bumped the tick to five seconds and the range to 2,000 and then 5,000 blocks as "an unconfirmed, conservative guess", and both were reverted the same day to the brief's documented values. The `MAX_RANGE` comment even quotes the brief's own anticipation that 500 blocks might prove impractical given Seaport's volume on Polygon, "which is exactly what happened", and asks future readers to revisit it rather than treat 500 as sacred. The range shrinking logic below is the code's answer to that.

At steady state the loop processes about seven or eight new blocks per tick (fifteen seconds of two second blocks). When catching up after downtime, it processes up to 500 blocks per tick, so it gains on the chain at roughly 33 blocks per second of wall clock versus the chain's 0.5, and a day of downtime (about 43,000 blocks) is recovered in roughly twenty two minutes if nothing gets rate limited.

### Construction

```ts
// src/components/marketplacev2/poller/tick/chain-event-source-poller.service.ts
this.provider = new JsonRpcProvider(this.chainConfig.polygonRpcUrl, MARKETPLACEV2_CHAIN_ID, { staticNetwork: true });
this.eventTopics = assertEventTopics((message) => this.customLoggerService.debug(message));
```

`staticNetwork: true` is an ethers version 6 option that tells the provider to trust the network you passed (chain 137) instead of calling `eth_chainId` to detect it. The comment explains the real motivation: without it, an unreachable RPC makes ethers retry network detection every second forever, flooding logs. Each of the poller's three classes that talk to the chain (this one, `SeaportEventApplicationService` for receipts, `DomainEventApplicationService` for block timestamps) builds its own `JsonRpcProvider` against the same URL. `OrderService` builds a fourth. They are cheap objects, but they do not share ethers' internal request batching or any cache.

### start, stop, and the unhandled rejection guard

```ts
// src/components/marketplacev2/poller/tick/chain-event-source-poller.service.ts
async start(): Promise<void> {
    if (this.intervalHandle) return;
    this.intervalHandle = setInterval(() => {
        this.tick().catch((err) => {
            this.customLoggerService.error(`poller tick: unhandled failure - ${err instanceof Error ? err.message : String(err)}. Cursor not advanced, will retry next tick.`);
        });
    }, TICK_INTERVAL_MS);
    this.customLoggerService.log(`poller started - ticking every ${TICK_INTERVAL_MS}ms`);
}
```

Three design choices live in these lines. It uses a raw `setInterval` rather than `@nestjs/schedule`'s `@Interval` or `@Cron`, so the start can be conditional on the flag and the class owns its own lifecycle. `start()` is idempotent, guarded by `intervalHandle`. And the `.catch` on `tick()` is load bearing: since Node 15, an unhandled promise rejection terminates the process, so a single failed cursor read (which is not wrapped inside `runTick`) would otherwise take the entire API down. The first tick fires fifteen seconds after boot, not immediately, which conveniently guarantees the cursor seed has finished.

### The concurrency guard

```ts
// src/components/marketplacev2/poller/tick/chain-event-source-poller.service.ts
async tick(): Promise<void> {
    if (this.tickInFlight) {
        this.customLoggerService.debug('poller tick skipped - previous tick still in flight');
        return;
    }
    this.tickInFlight = true;
    try {
        await this.runTick();
    } finally {
        this.tickInFlight = false;
    }
}
```

`setInterval` does not wait for the previous callback's promise. If a tick takes longer than fifteen seconds (a slow RPC, retries with backoff, a catch up range with hundreds of logs each doing database work), the next interval fires while the first is still running. This boolean makes the second one skip instead of running two ticks against the same cursor. Because JavaScript is single threaded and the check and set happen synchronously before the first `await`, there is no race inside one process.

It is important to see how small the protection is. It is an instance field on one object in one Node process. It does nothing across processes, containers or machines. Compare the v1 transaction cron in `03/04`, which solved its overlap problem with a database level compare and swap claim (`UPDATE ... WHERE reconciliationStatus = 'Pending' RETURNING id`) precisely so that it would be safe across overlapping runs. The poller relies instead on idempotent writes, discussed in `09`, plus a policy that only one process should ever poll.

### runTick, step by step

```ts
// src/components/marketplacev2/poller/tick/chain-event-source-poller.service.ts
private async runTick(): Promise<void> {
    this.lastTickAt = new Date();

    const lastProcessedBlock = await this.readCursor();
    const from = lastProcessedBlock + 1;

    let head: number;
    try {
        head = await retryWithBackoff(() => this.provider.getBlockNumber(), { isRetryable: isRetryableChainError });
    } catch (err) {
        this.customLoggerService.error(`poller tick: failed to fetch chain head - ${(err as Error).message}. Cursor not advanced, will retry next tick.`);
        return;
    }

    const to = Math.min(from + MAX_RANGE - 1, head - CONFIRMATIONS);
    if (to < from) {
        this.customLoggerService.debug(`poller tick: nothing new yet (from=${from}, head=${head}, confirmations=${CONFIRMATIONS})`);
        this.lastTickCounts = { seaportLogs: 0, domainLogs: 0, matched: 0 };
        return;
    }
    ...
```

1. Stamp `lastTickAt` first, before anything can fail, so the health endpoint can tell "the process is alive but failing" apart from "the process stopped ticking".
2. Read the cursor. `readCursor` throws if the row is missing, which can only mean boot ordering broke; the throw escapes `runTick` and is caught by the `.catch` in `start()`.
3. Fetch the chain head with retries (details below). On failure, log at error level and return without touching anything.
4. Compute the inclusive range. `from + MAX_RANGE - 1` is the 500 block cap; `head - CONFIRMATIONS` is the safety buffer. Whichever is smaller wins. When caught up, `to` is usually a handful of blocks above `from`. If the chain has not yet produced enough new blocks to clear the buffer, `to < from` and the tick is a no op that zeroes the per tick counts.

Then it fetches logs.

```ts
// src/components/marketplacev2/poller/tick/chain-event-source-poller.service.ts
    let seaportLogs: Log[];
    let domainLogs: Log[];
    let effectiveTo: number;
    try {
        ({ seaportLogs, domainLogs, to: effectiveTo } = await this.fetchLogsWithRangeShrink(from, to));
    } catch (err) {
        this.customLoggerService.warn(`poller tick: getLogs failed for range [${from}, ${to}] - ${(err as Error).message}. Cursor not advanced, will retry next tick.`);
        return;
    }

    this.customLoggerService.debug(`poller tick: range [${from}, ${effectiveTo}], seaportLogs=${seaportLogs.length}, domainLogs=${domainLogs.length}`);

    const blockTimestamps = await this.fetchFillBlockTimestamps(seaportLogs);
```

5. Fetch both log sets for the same range, possibly shrinking it. If that fails after retries, warn and return. Note the level: this is `warn`, and in this codebase's logger `warn` never reaches CloudWatch (see the logging section), so a poller stuck on `getLogs` failures is visible only in container stdout.
6. Log the range at debug level. In commit `79a63770` this line was bumped to `log` level with a comment explaining that `debug()` is written nowhere visible; commit `bdd8e80a` moved it back to `debug` because it fires every fifteen seconds, and a spec now asserts it is debug. The result is that, in production, there is no visible per tick heartbeat line at all.
7. Fetch block timestamps for fills, best effort.

Then it applies every log.

```ts
// src/components/marketplacev2/poller/tick/chain-event-source-poller.service.ts
    let matched = 0;
    const mismatchedFillTxHashes = new Set<string>();
    for (const log of seaportLogs) {
        try {
            if (await this.applySeaportLog(log, blockTimestamps, mismatchedFillTxHashes)) matched++;
        } catch (err) {
            if (!isDeterministicLogError(err)) {
                this.customLoggerService.error(`${POLLER_TRANSIENT_APPLY_FAILURE_MARKER} on seaport log (tx ${log.transactionHash}, block ${log.blockNumber}) - ${(err as Error).message}. Cursor not advanced, will retry the range next tick.`);
                return;
            }
            if (log.topics[0] === this.eventTopics.orderFulfilled) mismatchedFillTxHashes.add(log.transactionHash);
            this.customLoggerService.error(`poller tick: failed applying seaport log (tx ${log.transactionHash}, block ${log.blockNumber}) - ${(err as Error).message}. Deterministic failure, skipping this log, continuing the tick.`);
        }
    }
    for (const log of domainLogs) {
        // same shape, calling applyDomainLog
    }

    this.lastTickCounts = { seaportLogs: seaportLogs.length, domainLogs: domainLogs.length, matched };
    await this.advanceCursor(effectiveTo);
}
```

8. Apply every Seaport log first, in node order, then every domain log. This ordering is a correctness device: a sale emits both an `OrderFulfilled` and a `Transfer` in the same transaction, and processing the fill first is what stops the transfer from relabelling a sold order as invalid. `09` covers exactly how, and also the limits of "all Seaport logs before all domain logs" as an ordering guarantee across different blocks.
9. Each log is applied in its own try and catch, and the catch classifies the error (next section). A transient failure aborts the tick without moving the cursor. A deterministic failure is logged and skipped, and if it was a fill, its transaction hash goes into `mismatchedFillTxHashes` so the paired `Transfer` cannot invalidate the order.
10. Record counts and advance the cursor to `effectiveTo`, which may be smaller than the `to` computed in step 4 if the range was shrunk.

`advanceCursor` is the final, separate write.

```ts
// src/components/marketplacev2/poller/tick/chain-event-source-poller.service.ts
private async advanceCursor(to: number): Promise<void> {
    await this.cursorRepository.update({ id: SINGLETON_CURSOR_ID }, { lastProcessedBlock: String(to), updatedAt: new Date() });
}
```

This is the heart of the durability model. The log applications each commit their own small transaction, and the cursor update is a separate statement afterwards. If the process dies after some logs are applied but before the cursor moves, the next tick replays the whole range. That is "at least once" delivery, and it is only correct because every application is idempotent, which is the central claim `09` examines event by event. The reverse failure, the cursor moving past a log that was never applied, can only happen through the deliberate deterministic skip path, or through the multi process scenarios described below.

### Range shrinking

```ts
// src/components/marketplacev2/poller/tick/chain-event-source-poller.service.ts
private async fetchLogsWithRangeShrink(from: number, to: number): Promise<{ seaportLogs: Log[]; domainLogs: Log[]; to: number }> {
    let currentTo = to;
    for (;;) {
        try {
            const [seaportLogs, domainLogs] = await Promise.all([
                retryWithBackoff(() => this.provider.getLogs({
                    address: this.chainConfig.seaportAddress,
                    topics: [[this.eventTopics.orderFulfilled, this.eventTopics.orderCancelled, this.eventTopics.counterIncremented]],
                    fromBlock: from,
                    toBlock: currentTo
                }), { isRetryable: isRetryableChainError }),
                retryWithBackoff(() => this.provider.getLogs({
                    address: this.chainConfig.domainNftAddress,
                    topics: [[this.eventTopics.transfer, this.eventTopics.approvalForAll]],
                    fromBlock: from,
                    toBlock: currentTo
                }), { isRetryable: isRetryableChainError })
            ]);
            return { seaportLogs, domainLogs, to: currentTo };
        } catch (err) {
            if (isRangeTooLargeError(err) && currentTo > from) {
                const shrunkTo = from + Math.floor((currentTo - from) / 2);
                this.customLoggerService.error(`poller tick: RPC rejected block range [${from}, ${currentTo}] as too large - shrinking to [${from}, ${shrunkTo}] and retrying`);
                currentTo = shrunkTo;
                continue;
            }
            throw err;
        }
    }
}
```

The filter syntax is worth decoding. `topics: [[a, b, c]]` means "position zero must be any one of a, b or c", an OR inside the first topic slot. There is no filter on `offerer` or anything else, so the Seaport query returns every fill, cancel and counter bump on the entire Seaport contract on Polygon, across every marketplace that uses it. That is a lot of logs, which is precisely why 500 block ranges get rejected and why the shrink exists. Filtering on `offerer` is not possible in general because the poller would need to know every maker address in advance, but it is worth knowing that the overwhelming majority of what this call downloads is somebody else's business.

Both queries always run with the same `currentTo`, so the cursor stays consistent for both contracts. When either one comes back with a "range too large" style error, both are retried with the upper bound halved. The halving can go all the way down to a single block. If even a one block range is rejected, the error is thrown, the tick warns and returns, and the poller is stuck on that block forever; that is extremely unlikely on Polygon but it has no escape hatch.

The shrink only applies to the current tick. The next tick computes a fresh 500 block range from the new cursor, so on a stretch of chain that is consistently too dense, every tick pays for one or more rejected requests before it finds a size that works. Remembering the last successful size would avoid that.

`isRangeTooLargeError` is a heuristic over provider error shapes.

```ts
// src/components/marketplacev2/poller/tick/chain-event-source-poller.service.ts
function isRangeTooLargeError(err: unknown): boolean {
    const e = err as { code?: unknown; shortMessage?: unknown; message?: unknown; info?: { error?: { code?: unknown; message?: unknown } }; error?: { code?: unknown; message?: unknown } } | null | undefined;
    if (!e) return false;
    const rpcCode = e.info?.error?.code ?? e.error?.code;
    if (rpcCode === -32005) return true;
    const message = [e.shortMessage, e.message, e.info?.error?.message, e.error?.message]
        .filter((part): part is string => typeof part === 'string')
        .join(' ')
        .toLowerCase();
    return message.includes('range') && (message.includes('large') || message.includes('limit') || message.includes('too many') || message.includes('10000') || message.includes('10,000'));
}
```

ethers version 6 wraps JSON RPC errors it does not recognise into an `UNKNOWN_ERROR` whose `error` property holds the original `{ code, message }` from the node, which is why it digs into `e.error` and `e.info.error`. Code `-32005` is the "limit exceeded" code Infura style providers use. The message fallback looks for "range" together with one of several limit words. It is reasonable, but it is string matching on third party error text, and there is no spec test for it or for the shrink loop at all (a grep of every spec in the folder for `-32005`, "too large" and "shrink" finds nothing). That is the most important untested code path in the tick loop, because it is exactly the path the comment says already fired in UAT.

### Block timestamps for fills

```ts
// src/components/marketplacev2/poller/tick/chain-event-source-poller.service.ts
private async fetchFillBlockTimestamps(seaportLogs: Log[]): Promise<Map<number, number>> {
    const blockNumbers = [...new Set(seaportLogs.filter((log) => log.topics[0] === this.eventTopics.orderFulfilled).map((log) => log.blockNumber))];
    const blockTimestamps = new Map<number, number>();
    if (blockNumbers.length === 0) {
        return blockTimestamps;
    }
    try {
        const blocks = await Promise.all(blockNumbers.map((blockNumber) => retryWithBackoff(() => this.provider.getBlock(blockNumber), { isRetryable: isRetryableChainError })));
        for (const block of blocks) {
            if (block) blockTimestamps.set(block.number, block.timestamp);
        }
    } catch (err) {
        this.customLoggerService.warn(`poller tick: failed to fetch block timestamps for fill events - ${(err as Error).message}. filledAt will fall back to processing time this tick.`);
    }
    return blockTimestamps;
}
```

The intent is good: `filledAt` should be when the sale happened on chain, not when the poller got around to it, and those diverge by hours during catch up. The implementation has a real cost problem. It fetches a block for every distinct block containing any `OrderFulfilled` log, before checking whether any of those fills is ours. Since Seaport on Polygon carries every marketplace's fills, a 500 block catch up range can easily contain fills in hundreds of distinct blocks, which means hundreds of concurrent `eth_getBlockByNumber` calls fired by `Promise.all`, almost all of them for other marketplaces' sales. ethers version 6 will coalesce concurrent calls into JSON RPC batches (by default up to 100 requests per batch, gathered over a 10 ms window), which softens the burst, but the paid RPC still bills every call. And because it is a single `Promise.all`, one block failing after its retries throws away every timestamp for the tick, so every fill in that tick falls back to processing time. Filtering to fills whose `orderHash` exists in `tbl_marketplacev2_orders` first (one `WHERE "orderHash" IN (...)` query) would cut this to almost nothing.

### Dispatch by topic

```ts
// src/components/marketplacev2/poller/tick/chain-event-source-poller.service.ts
private async applySeaportLog(log: Log, blockTimestamps: Map<number, number>, mismatchedFillTxHashes: Set<string>): Promise<boolean> {
    const enrichedLog = { topics: log.topics, data: log.data, transactionHash: log.transactionHash, blockNumber: log.blockNumber, blockTimestamp: blockTimestamps.get(log.blockNumber) };
    switch (log.topics[0]) {
        case this.eventTopics.orderFulfilled:
            return this.seaportEventApplicationService.applyOrderFulfilled(enrichedLog, mismatchedFillTxHashes);
        case this.eventTopics.orderCancelled:
            return this.seaportEventApplicationService.applyOrderCancelled(log);
        case this.eventTopics.counterIncremented:
            return this.seaportEventApplicationService.applyCounterIncremented(log);
        default:
            return false;
    }
}
```

Only the fill gets the enriched log with a timestamp. `OrderCancelled` is passed the raw `log`, which has no `blockTimestamp`, so its `occurredAt` always falls back to processing time. `09` covers the consequence. The `switch` compares lowercase hex strings; ethers returns topics lowercase and `topicHash` is lowercase, so the comparison is safe.

## Retry and backoff

Two small shared utilities in `src/@core/utils/retry/` wrap every RPC call the poller makes.

```ts
// src/@core/utils/retry/retry-with-backoff.util.ts
const DEFAULT_IS_RETRYABLE = (error: any): boolean => error?.code === 429 || error?.status === 'RESOURCE_EXHAUSTED' || error?.response?.status === 429;

export async function retryWithBackoff<T>(fn: () => Promise<T>, options: RetryOptions = {}): Promise<T> {
    const { maxRetries = 3, baseDelayMs = 500, isRetryable = DEFAULT_IS_RETRYABLE } = options;
    let attempt = 0;
    for (;;) {
        try {
            return await fn();
        } catch (error) {
            if (!isRetryable(error) || attempt >= maxRetries) {
                throw error;
            }
            const jitter = Math.random() * baseDelayMs;
            await new Promise((resolve) => setTimeout(resolve, baseDelayMs * 2 ** attempt + jitter));
            attempt++;
        }
    }
}
```

With the defaults, a call that keeps failing with a retryable error is attempted four times in total (the first plus three retries), sleeping roughly 500 to 1,000 ms, then 1,000 to 1,500 ms, then 2,000 to 2,500 ms between them, so about five seconds of backoff before giving up. The jitter (a random amount up to one base delay) spreads out retries from concurrent callers so they do not all hammer the RPC at the same instant. A non retryable error is rethrown immediately with no delay. The default predicate only recognises quota errors (HTTP 429 and gRPC style `RESOURCE_EXHAUSTED`), which suits quota limited HTTP APIs, so every poller call site overrides it.

```ts
// src/@core/utils/retry/is-retryable-chain-error.util.ts
export function isRetryableChainError(error: unknown): boolean {
    const err = error as { code?: unknown; status?: unknown; response?: { status?: unknown } } | null | undefined;
    if (!err) return false;
    const code = typeof err.code === 'string' ? err.code : undefined;
    if (code && ['ECONNRESET', 'ETIMEDOUT', 'ECONNREFUSED', 'ENOTFOUND', 'TIMEOUT', 'NETWORK_ERROR', 'SERVER_ERROR'].includes(code)) {
        return true;
    }
    const status = typeof err.status === 'number' ? err.status : typeof err.response?.status === 'number' ? err.response.status : undefined;
    return typeof status === 'number' && status >= 500 && status < 600;
}
```

This covers Node socket errors, ethers version 6's own `TIMEOUT`, `NETWORK_ERROR` and `SERVER_ERROR` codes, and any 5xx status. It was extracted from `order.service.ts` so the poller and the order validation path share one predicate. It does not include 429, so the custom predicate actually loses the default's rate limit handling; in practice ethers version 6's `FetchRequest` already retries HTTP 429 internally with its own throttle before the error ever surfaces, so this is less dangerous than it looks, but it is a subtle dependency on library behaviour. It also does not treat ethers' `UNKNOWN_ERROR` wrapper as retryable, which means common transient node responses such as Polygon's `header not found` (a load balanced node that has not seen the block yet) fail the tick immediately and wait fifteen seconds for the next one, which is an acceptable outcome.

## Error classification: deterministic versus transient

This is the newest piece of the loop (commit `bdd8e80a`, 1 October 2026), and its history is the best lesson in this folder. The very first version wrapped the whole range in one try and catch: any failure aborted the tick and the cursor never moved, so a single bad log stalled the poller forever. Commit `96f221ec` switched to a per log try and catch that logged and skipped every failure, which unstuck the poller but meant a database timeout during a fill would permanently lose that sale, and worse, the paired `Transfer` would then invalidate the order that had actually sold. The current version splits the two cases.

```ts
// src/components/marketplacev2/poller/tick/chain-event-source-poller.service.ts
function isDeterministicLogError(err: unknown): boolean {
    const e = err as { code?: unknown; driverError?: { code?: unknown } } | null | undefined;
    if (!e) return false;
    // ethers decode/validation failures
    if (e.code === 'BAD_DATA' || e.code === 'INVALID_ARGUMENT' || e.code === 'NUMERIC_FAULT') return true;
    // Postgres class 22 (data exception) and 23 (integrity constraint violation)
    const pgCode = e.driverError?.code ?? e.code;
    return typeof pgCode === 'string' && /^2[23]/.test(pgCode);
}

export const POLLER_TRANSIENT_APPLY_FAILURE_MARKER = 'poller tick: TRANSIENT apply failure';
```

Deterministic means "this exact log will fail the same way every time", so skipping it is the only way forward. The recognised shapes are ethers' value errors (`INVALID_ARGUMENT` is what `getAddress` throws on a malformed address) and Postgres SQLSTATE classes 22 and 23, which TypeORM surfaces on `QueryFailedError` via `driverError.code`. Everything else, including any plain `Error` with no code at all, is treated as transient: the tick logs the stable marker string (intended for a CloudWatch metric filter and alarm) and returns without advancing the cursor, so the next tick retries the whole range.

That default is the safe direction for data, and it has a cost worth naming. Any bug that throws a code less error, a `TypeError` from an unexpected `null`, a Postgres class 42 error (`42703 undefined_column`) after a schema drift or a missed migration, a block that `fetchBlockTimestamp` reports as not found, is classified as transient and wedges the poller on that range permanently. Every fifteen seconds it retries, fails, logs the marker, and the lag climbs. The alarm on the marker is what turns that from silent into noticed, and there is no evidence in the repo (no Terraform metric filter for the string) that the alarm actually exists yet. A retry counter per range that eventually escalates or dead letters the log would bound the damage.

The deterministic skip path has the opposite cost. When it fires, the event is gone for good: there is no dead letter table, no record beyond the error log line. A skipped `Transfer` or `ApprovalForAll` leaves a ghost listing active; a skipped fill leaves a sold domain listed. The `mismatchedFillTxHashes` trick protects the specific case of a skipped fill being turned into an `invalid` by its own transfer, which is good, but nothing reconciles the skipped event later.

Note also that `decode()` in both application services catches its own parse errors and returns `null`, so a malformed log never actually reaches this classifier as `BAD_DATA`; it simply returns `false` ("not matched") and is skipped silently apart from an error log. The ethers codes in the classifier therefore mostly matter for `getAddress` calls on decoded values.

## Health endpoints, guards, and what they report

```ts
// src/components/marketplacev2/poller/poller.controller.ts
@Controller('marketplacev2/poller')
export class PollerController {
    @Get('_health')
    @UseGuards(AccessTokenGuard, AdminTokenGuard)
    async health(): Promise<Response> {
        return new Response('marketplacev2 poller health check', await this.pollerService.health());
    }

    @Get('health')
    @UseGuards(AccessTokenGuard)
    async status(): Promise<Response> {
        return new Response('marketplacev2 poller lag and status', await this.chainEventSource.getStatus());
    }
}
```

With the global `/api/v1` prefix the routes are as follows.

| Method and path | Guards | Returns |
|---|---|---|
| `GET /api/v1/marketplacev2/poller/_health` | `AccessTokenGuard` (Bearer JWT) and `AdminTokenGuard` (`admin-token` header equal to `ADMIN_TOKEN`) | `{ module: 'poller', status: 'ok', cursor: { lastProcessedBlock, updatedAt } \| null }`, the sprint one diagnostic |
| `GET /api/v1/marketplacev2/poller/health` | `AccessTokenGuard` only | The `PollerStatus` object below |

Both use the house controller pattern, a thin dispatcher wrapping the service result in `new Response(message, data)`. The two guards on `_health` read different headers (`Authorization` for the JWT, `admin-token` for the static token), so a caller must present both. `src/components/marketplacev2/health-routes.guards.spec.ts` pins this with reflection on `GUARDS_METADATA`, asserting `_health` has exactly `[AccessTokenGuard, AdminTokenGuard]` and `health` has exactly `[AccessTokenGuard]`.

`AdminTokenGuard` itself is shared and older than this folder, and it has two weaknesses worth knowing because `_health` now depends on it: it compares with loose `==`, and it compares against `configService.get('ADMIN_TOKEN')`. If that config value were ever undefined in some environment, a request with no `admin-token` header at all would satisfy `undefined == undefined` and pass. The comparison is also not constant time. Neither is catastrophic for a diagnostic route that still needs a valid JWT, but the guard is used elsewhere too.

`getStatus` builds the real health picture.

```ts
// src/components/marketplacev2/poller/tick/chain-event-source-poller.service.ts
async getStatus(): Promise<PollerStatus> {
    const [lastProcessedBlock, chainHead] = await Promise.all([
        this.readCursor(),
        this.provider.getBlockNumber().catch((err) => {
            this.customLoggerService.error(`poller getStatus: failed to fetch chain head - ${err instanceof Error ? err.message : String(err)}`);
            return null;
        })
    ]);
    const lagBlocks = chainHead === null ? null : Math.max(0, chainHead - lastProcessedBlock);
    const tickIsFresh = this.lastTickAt !== null && Date.now() - this.lastTickAt.getTime() <= STALE_TICK_MS;
    return {
        lastProcessedBlock,
        chainHead,
        lagBlocks,
        lastTickAt: this.lastTickAt ? this.lastTickAt.toISOString() : null,
        healthy: chainHead !== null && tickIsFresh && lagBlocks <= UNHEALTHY_LAG_BLOCKS,
        counts: { ...this.lastTickCounts }
    };
}
```

`healthy` requires three things together: the RPC answered, a tick was attempted within the last 45 seconds (three intervals), and the cursor is within 300 blocks (about ten minutes) of the head. The RPC failure is caught separately so a down node yields `healthy: false` with `chainHead: null` instead of a 500, which, as the comment says, is exactly the moment the endpoint most needs to work. Because `lagBlocks` is measured against the raw head, a perfectly caught up poller always shows a lag of at least `CONFIRMATIONS` (five). The counts reflect only the last completed tick, and are not updated when a tick aborts early on a transient failure, so a wedged poller keeps showing the counts from its last good tick.

Some practical consequences. `lastTickAt` and `lastTickCounts` are in memory, per process. Behind a load balancer with more than one task, two consecutive calls can hit different processes and see different answers, and a process with the poller disabled always reports unhealthy. `readCursor` can still throw (database down), which surfaces as a 500. Every call costs one database read and one paid RPC call, the route is open to any logged in user, and there is no throttle on it, so a script with a normal user's JWT can burn RPC quota at will. And `getLag()`, required by the interface, is not called anywhere outside its own spec.

## Logging: where the poller's messages actually go

This matters more than it sounds, because the poller's whole failure model is "log loudly and let a human notice".

```ts
// src/logger/file-logge.ts
log(message: string, trace?: string) {
    this.logger.info({ context: this.context, message });
    console.log(new Date().toISOString(), this.context, message, trace, 'log')
}
error(message: string, trace?: string) {
    this.logger.error({ context: this.context, message });
    console.log(new Date().toISOString(), this.context, message, trace, 'error')
}
warn(message: string) {
    this.logger.warn({ context: this.context, message });
    console.log(new Date().toISOString(), this.context, message, 'warn')
}
debug(message: string) {
    this.logger.debug({ context: this.context, message });
}
```

The winston logger has two CloudWatch transports, one at level `error` (which in winston means error only, since error is the most severe level) and one at level `info` with a format filter that drops anything whose level is not exactly `info`. So `error()` reaches the error group and stdout, `log()` reaches the info group and stdout, `warn()` reaches stdout only (no CloudWatch group accepts `warn`), and `debug()` reaches nowhere at all. The comment the author left in commit `79a63770` and later removed said the same thing.

Map that onto the poller. Visible in CloudWatch: chain head failures, transient apply failures with the marker, deterministic skips, range shrinks, verification mismatches, state transitions logged with `log`. Visible only in container stdout: `getLogs` failures for a range, fill timestamp fetch failures, the "chain wins" transitions from cancelled, invalid or expired to filled, `CounterIncremented` alerts, fills recorded without gas info, and the "transfer accompanied a failed verification" warning. Visible nowhere: the per tick range line, skipped ticks, "nothing new yet", and the boot time topic assertion summary.

There is also a latent boot race. `CustomLoggerService`'s constructor calls `initializeLogger()` without awaiting it, and `initializeLogger` fetches a secret before assigning `this.logger`. Until that fetch resolves, `this.logger` is `undefined`, and any call to `debug()` throws a `TypeError`. Both `PollerService.onModuleInit` and the `ChainEventSourcePoller` constructor call `assertEventTopics` with a callback that calls `debug()` synchronously. In practice the logger's secret fetch usually finishes first, because `ChainConfigModule`'s factory also does a Secrets Manager round trip before the poller can be constructed, but nothing guarantees that ordering. If the logger's fetch is ever slower, the boot fails with a `TypeError` that looks completely unrelated to the poller.

## The multiple process problem, honestly

The design statement is "single instance only", and it is enforced by configuration, not by code. Here are the ways two pollers can end up running against the same database on this branch.

| Scenario | Where it comes from | What happens |
|---|---|---|
| Rolling deploy on ECS | `terraform/modules/ecs/main.tf` sets `desired_count = var.TASK_COUNT` (default 1) with `deployment_maximum_percent` defaulting to 200 and minimum healthy 100 | During every deploy the new task starts and becomes healthy before the old one stops, so for minutes two tasks with the same secret both poll |
| Scaling out | `TASK_COUNT` above 1, or a future autoscaling policy | Permanent double polling |
| A developer's machine | Local boot pointed at a shared secret where `POLLER_ENABLED` is `true`, or at a secret missing the key while `NODE_ENV=production` | The exact 29 September incident |
| Fallback misfire | Secret missing `POLLER_ENABLED` on any server process | Every server process polls, because PM2 sets `NODE_ENV=production` |

PM2 itself runs one forked process (no `instances` key in `pm2.config.js`), so cluster mode is not a risk today.

What actually goes wrong when two pollers overlap? Correctness of the order state mostly survives, because every write is a conditional `UPDATE` (Postgres row locking plus `READ COMMITTED` re evaluation of the `WHERE` makes the second writer see zero affected rows) and every insert into the history and transaction tables is `ON CONFLICT DO NOTHING` against a unique key. `09` walks through why. What breaks is the cursor. Process A reads cursor 1,000 and gets stuck in RPC retries; process B reads 1,000, processes to 1,500, then to 2,000; A finally finishes and writes 1,500. The cursor has moved backwards by 500 blocks, the lag on the health endpoint jumps, and all of that range is reprocessed, wasting RPC quota and, because writes are idempotent, mostly nothing else. That is exactly "the cursor jumped backward and the lag never dropped" from the incident comment. Side effects that are not idempotent (duplicate `log` lines, duplicate `fetchGasInfo` and `getBlock` calls) double up.

The fix is cheap and well known. Either make `advanceCursor` a compare and swap (`UPDATE ... SET "lastProcessedBlock" = :to WHERE id = 1 AND "lastProcessedBlock" = :from`, and abandon the tick if zero rows were affected), or take a Postgres advisory lock (`pg_try_advisory_lock`) at the start of each tick and skip the tick if another process holds it. Either would make the "single instance" rule self enforcing instead of a matter of configuration discipline.

## Re orgs, finality, and what five blocks buys

Five confirmations on Polygon is about ten seconds. Polygon PoS has historically had re orgs much deeper than that (a widely reported one in February 2023 was over 150 blocks), and while the newer milestone based finality has made deep re orgs far rarer, a fixed count of five is a thin margin. The consequences of a re org deeper than the buffer are asymmetric. A fill that is re orged out leaves the order `filled`, which the code treats as terminal and never revisits, plus an append only `SALE` row in `tbl_marketplacev2_transactions` and a `SOLD` row in listing history, none of which can be corrected by the poller. A `Transfer` or revocation that is re orged out leaves the order `invalid` when it is actually fillable, which is recoverable only if somebody later fills it. Since the cursor never rewinds, the replacement branch's logs for those blocks are never read either.

The simplest structural improvement is to replace `head - CONFIRMATIONS` with the chain's own finality signal: ethers version 6 supports `provider.getBlock('finalized')`, and Polygon nodes expose a `finalized` tag backed by milestones. That would cost a little latency and remove most of the residual risk without writing any re org rollback logic.

## Comparison with the older mechanisms

File `05/09` showed that the v1 `listener` module, despite its name, is a dynamic cron job registry built on `SchedulerRegistry`, wired into `domain-order.service.ts` and never called. File `03/04` showed the v1 transaction cron, an every minute `@Cron` that reconciles rows the frontend created in a `Pending` state by fetching each transaction's receipt. The new poller is a third, genuinely different design, and lining them up makes each one clearer.

| Question | v1 `listener` (`05/09`) | v1 transaction cron (`03/04`) | v2 poller (this folder) |
|---|---|---|---|
| What triggers work | Nothing currently; a caller would register a named `CronJob` | `@Cron(EVERY_MINUTE)` from `@nestjs/schedule` | A raw `setInterval` every 15 s, started only if `POLLER_ENABLED` |
| How it learns what happened | Not implemented | The frontend submits a tx hash; the cron asks for that one receipt | Reads every relevant log from the chain itself; no tx hash from anyone |
| What it can discover | n/a | Only transactions the backend was told about | Anything on chain: sales made from other UIs, on chain cancels, transfers, approval revocations |
| Unit of progress | n/a | Each pending row, with `retryCount` and a seven miss give up | A block range, with one durable cursor |
| Overlap protection | n/a | Database compare and swap claim on `reconciliationStatus` | In process boolean only, plus idempotent writes |
| Pending state in the database | `PROCESSING` / `COMPLETED` rows | Yes, rows start `Pending` and are reconciled | None; rows are only ever written after the event is past the confirmation buffer |
| Re org handling | n/a | Receipt status at the time of the check, no depth | Five block buffer, no rollback |
| Chains | n/a | Five Web3 instances (BNB, ETH, Polygon, Arbitrum, Base) | Polygon only (chain 137 hardcoded) |
| Library | `cron` | `web3` 1.x | `ethers` 6 |
| Verification of the event | n/a | Contract address check, revert check, owner check | Full decode and comparison against the stored signed order (see `09`) |

The biggest conceptual shift is in the third row. The v1 cron is reactive to the backend's own requests: if a sale happens and nobody tells the backend the hash, it never finds out. The poller is reactive to the chain: a Seaport order signed on this marketplace but filled through some aggregator, or cancelled from another marketplace's UI, is still seen, because the signed order hash is what links the on chain event to the database row. That is why v2 can have a `transactions` table with "deliberately no status/retryCount/reconciliationStatus column", as its entity comment puts it: nothing is ever inserted in a pending state, so there is nothing to reconcile. Both v1 mechanisms are still present on `uat` (`TransactionCheckCronModule` is still imported in `app.module.ts`), they simply serve the v1 marketplace.

The one place v1 is stronger is overlap safety. The v1 cron's claim pattern is safe under any number of overlapping runs or processes; the poller's guard is per process, and its cursor write is not conditional.

## Risks and bugs found in this layer

| Where | Risk | Concrete failure |
|---|---|---|
| `tick/chain-event-source-poller.service.ts:388-390` | Unconditional cursor write, no compare and swap or lock | Two overlapping pollers (rolling ECS deploy at 200 percent, a dev laptop) move the cursor backwards, as in the 29 September incident |
| `tick/chain-event-source-poller.service.ts:80, 182-193` | Concurrency guard is an in memory boolean | Does nothing across processes or containers |
| `terraform/modules/ecs/main.tf:539` with `variables.tf` `deployment_max_percent = 200` | Every deploy runs two tasks briefly | Two pollers on the same secret for the length of the deploy |
| `config/chain-config.loader.ts:45-49` | Missing `POLLER_ENABLED` falls back to `NODE_ENV === 'production'` | Every server process polls if the key is ever dropped from a secret |
| `config/chain-config.loader.ts:20, 58-63` | Silent fallback start block 93,840,000 | A poller booted on the fallback never sees events before that block, and since the seed is `ON CONFLICT DO NOTHING`, fixing the secret later does not move it |
| `poller.service.ts:37` | Seed never updates an existing row | Changing `POLLER_START_BLOCK_POLYGON` after first boot has no effect without manual SQL |
| `constants/poller.constants.ts:22`, `tick/...:211` | Five blocks is the only re org protection | A deeper re org leaves a permanent `filled` order plus append only `SALE` and `SOLD` rows for a sale that never happened |
| `tick/...:53-61, 253-256, 265-268` | Unknown errors are "transient" | A code less bug, a class 42 SQL error, or a missing block wedges the poller on one range forever |
| `tick/...:257-258, 269` | Deterministic skips are permanent, no dead letter | A skipped `Transfer` or fill leaves a ghost listing that nothing ever reconciles |
| `tick/...:335-350` | Timestamp fetch for every fill on Seaport, not just ours, in one `Promise.all` | Hundreds of billed `getBlock` calls per catch up tick; one failure drops all timestamps so `filledAt` falls back to processing time |
| `tick/...:285-325` | Range shrink resets every tick and has no tests | Dense stretches pay for rejected requests every tick; a single block still too large stalls forever |
| `tick/...:30-40` | Error text heuristics | A provider wording change silently disables the shrink, stalling the poller on a range |
| `tick/...:224`, `src/logger/file-logge.ts:85-92` | `warn` never reaches CloudWatch, `debug` goes nowhere | A poller stuck on `getLogs` failures is invisible in CloudWatch; no heartbeat line at all in production |
| `src/logger/file-logge.ts:24-27, 37`, `poller.service.ts:34`, `tick/...:102` | Logger initialises asynchronously while the topic assertion calls `debug()` at construction | If the logger's secret fetch is slower than chain config's, boot fails with an unrelated looking `TypeError` |
| `abi/assert-event-topics.ts:25-29` | `OrderCancelled` and `CounterIncremented` topics not pinned | A typo in either fragment makes the poller silently stop seeing on chain cancels |
| `poller.service.ts:34` | Topic mismatch fails boot of the entire API | An ABI typo in this folder takes down every unrelated endpoint |
| `poller.controller.ts:41-46` | `health` open to any logged in user, unthrottled, one paid RPC call per request | A normal user's JWT in a loop burns RPC quota |
| `src/@core/common/guards/admin-token.guard.ts:16` | Loose `==` against a possibly undefined config value, not constant time | If `ADMIN_TOKEN` were unset, `_health` would accept a request with no admin header |
| `src/main.ts` (no `enableShutdownHooks`) | `onModuleDestroy` never runs on SIGTERM | Harmless thanks to cursor discipline, but `stop()` is effectively dead code in production |
| `entity/poller-cursor.entity.ts:24-34` | No `chainId`, no block hash, no owner column | A second chain needs a schema change; re orgs and competing writers cannot be detected |

## Frontend note

When a marketplace v2 screen shows a listing still "active" for half a minute after a buyer's wallet says the purchase succeeded, that is not a bug in the UI, it is this loop's fifteen second tick plus a five block buffer. The honest UX is to show the buyer an optimistic "purchase submitted, confirming on chain" state from the wallet's own receipt, and let the listing flip to sold when the next poll after confirmation lands. If you ever build an internal status page, `GET /api/v1/marketplacev2/poller/health` gives you `lagBlocks` and `lastTickAt`, and the distinction its comments draw is worth surfacing: a stale `lastTickAt` means the process stopped, while a large `lagBlocks` with a fresh `lastTickAt` means it is alive but falling behind or stuck retrying one range.
