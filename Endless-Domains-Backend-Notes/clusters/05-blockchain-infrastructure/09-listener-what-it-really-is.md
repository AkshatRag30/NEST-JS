# 09. Listener, What It Really Is

## The guess, stated honestly, before checking it

A folder named `listener`, sitting in a cluster full of blockchain code, invites one very specific guess, that it subscribes to live blockchain events, the way an `ethers.js` contract instance can call `.on('Transfer', callback)` to be notified in real time whenever a token moves, or whenever a mint happens. That is a completely reasonable guess to make from the name alone, and it is exactly the kind of guess file 01 of the wider notes for this project (`01-what-is-this-product.md`) warns you to check against the actual code rather than trust outright. So this file does that checking, and the guess turns out to be wrong.

## What the code actually contains

`listener.module.ts`, `listener.service.ts`, and `listener.repo.ts` contain no `ethers.js`, no `web3.js`, no contract, no ABI, and no reference to any blockchain event at all. What they do contain is `@nestjs/schedule`'s `SchedulerRegistry` and the `cron` package's `CronJob`:

```ts
import { SchedulerRegistry } from '@nestjs/schedule';
import { CronJob } from 'cron';
...
async startListener(listenerId: string, job: CronJob): Promise<void> {
    this.schedulerRegistry.addCronJob(listenerId, job);
    job.start();
}

async stopListener(listenerId: string): Promise<void> {
    const job = await this.schedulerRegistry.getCronJob(listenerId);
    job.stop();
    await this.schedulerRegistry.deleteCronJob(listenerId);
}
```

`SchedulerRegistry` is a NestJS facility for managing cron jobs dynamically, at runtime, rather than only through the static `@Cron(...)` decorator you might have seen used elsewhere for jobs that always run on a fixed schedule. `startListener` and `stopListener` let some other part of this codebase register a named job under a given `listenerId`, start it running on whatever schedule its own `CronJob` object was built with, and later find and stop that exact job by that same ID. That is a generic, reusable, dynamic job scheduler, not an event subscription mechanism.

The rest of the service persists a small database record for each named job:

```ts
@Entity({ name: 'tbl_listener' })
export class ListenerEntity extends BaseEntity {
    @Column({ nullable: false }) listenerId: string;
    @Column({ nullable: false, default: 'COMPLETED' }) status: string;
    @Column({ nullable: false, default: 'DOMAIN_ORDER' }) moduleName: string;
}
```

`ListenerStatus` is only ever `PROCESSING` or `COMPLETED`, and `saveListener`, `updateStatus`, and `getAllListenerByStatusAndModule` exist to let a caller record which named jobs currently exist, which module they belong to (the entity's own default value, `DOMAIN_ORDER`, is a strong hint about the intended caller), and whether each one is currently mid run or has finished. This is, in plain terms, a small, generic, database backed bookkeeping layer for dynamically managed cron jobs, tracking which ones exist, which module owns them, and whether each is currently busy.

## Confirming where it is actually wired in, and where it is not

Searching the rest of the codebase for who actually uses this service turns up exactly one consumer, `domain-order.service.ts`, which injects `ListenerServiceInterface` in its constructor:

```ts
@Inject('ListenerServiceInterface')
private readonly listenerService: ListenerServiceInterface,
```

Searching further, that injected `listenerService` is never actually called anywhere in `domain-order.service.ts`, no `startListener`, no `saveListener`, no `updateStatus`. The dependency is wired in, and nothing in this codebase currently calls it. This is a real, useful thing to be able to say plainly about a piece of infrastructure, `listener` is built, tested (it has its own service and repo, wired into a real module), and connected into the one feature (`DOMAIN_ORDER`, matching the entity's default) that would plausibly need it, but at the time of writing, it is not actually being exercised by any request path currently in this codebase. That is a completely different, and far more useful, finding than either "it definitely does X" or "it definitely does nothing," and it is only available because the actual code, not the folder name, was read.

## Why a domain order flow would plausibly want something like this at all

Even though it is not currently called, the shape of the tool makes sense once you connect it to what file 04 and the wider domain ordering flow (covered in its own cluster of notes) actually need. Confirming a blockchain transaction is not instant, blocks take time to be mined, and a backend that just fired off a mint or a claim cannot know immediately whether it succeeded. A natural way to handle that is a background job that wakes up periodically, checks the current on chain status of any pending domain orders, and updates their state once a transaction actually confirms, exactly the kind of job `startListener` would let a service spin up dynamically per order (or per batch), track through the `PROCESSING` and `COMPLETED` statuses this entity models, and tear down once it is done. Whether that is exactly what this team intended, or whether it was built for that purpose and later superseded by a different polling mechanism inside `domain-order.service.ts` itself, is not something the code alone can answer with certainty, and it would be a mistake to overstate confidence either way. What can be said with confidence is only what the code shows, a real, generic, working cron job registry, aimed at `DOMAIN_ORDER` by name, currently unused.

## The habit this file is really trying to model

The specific finding here, that `listener` is a cron job registry and not an event subscriber, matters less than the process that uncovered it. A name suggests a hypothesis, reading the actual service, repo, and entity either confirms or breaks that hypothesis, and searching for real callers across the rest of the codebase tells you whether the hypothesis, once corrected, is even something currently in active use. That three step habit, guess, read, then search for callers, is the one piece of this note worth carrying into every other unfamiliar module you meet in this codebase, blockchain related or not.
