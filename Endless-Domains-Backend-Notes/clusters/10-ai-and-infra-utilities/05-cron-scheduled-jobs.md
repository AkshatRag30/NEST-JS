# 05. Cron: what actually runs on a schedule, and how often

## The ScheduleModule, and where its jobs actually live

`app.module.ts` registers `ScheduleModule.forRoot()` once, near the top of its imports, which is what turns on NestJS's own `@Cron` decorator support application wide. That one registration is what lets any injectable class anywhere in this codebase declare a method that runs on a schedule, simply by decorating it, no further wiring needed per job. Because of that, the actual scheduled jobs are not all sitting in one obvious place, `CronModule` was written as the dedicated home for the general purpose ones, but `AiAlertService` from note 03 declares its own `@Cron` method too, entirely separately, since any provider anywhere in the module graph can do the same thing. One crucial caveat as of the October 2026 uat pull: `CronModule` is currently commented out of `app.module.ts` (line 177, commit `836d5f89`), so none of the `TasksService` jobs described in the next section are running at all on `uat`. A `@Cron` decorator only fires if its class is actually instantiated, and a provider is only instantiated if its module is imported somewhere. The update section at the end of this note lists exactly what stopped and what still runs.

## `TasksService`, the real cron job list

`src/components/cron/cron-service.ts` defines `TasksService`, and this is where nearly every recurring job in the whole application actually lives. Reading straight through its decorated methods in order:

`@Cron(CronExpression.EVERY_8_HOURS) handleCron()` finds every user who signed up but never verified their email, and for anyone who has been sent the verification email five times or fewer, sends it again and bumps their sent count. This is the mechanism that keeps nudging a half-finished signup toward completion without emailing the same person forever, the `sentEmailCount <= 5` cap is what stops it.

`@Cron(CronExpression.EVERY_HOUR) checkWalletBalance()` checks the ETH balance of the platform's own operational wallet address (`WALLET_ADDRESS`, an Infura backed websocket RPC connection) by calling `web3Value.eth.getBalance`, converts it to USD using a live price from the CoinGecko API through the shared `httpClient`, and if the USD value drops under 200 dollars, emails a low balance alert to `ankit@endlessdomains.io`. This is the kind of job that exists because the platform genuinely needs its own wallet funded, likely to cover gas fees for on chain operations it performs on a user's behalf, and running out of gas silently would be a real operational failure worth an automated alert.

`@Cron(CronExpression.EVERY_5_MINUTES) handlePendingOrderReminderCron()` delegates straight to `DomainOrderService.sendPendingOrderReminderEmail()`, nudging users who have an order sitting unfinished. Five minutes is a tight interval for a cron job, which suggests the underlying service itself is responsible for deciding who actually needs a reminder right now, rather than this job blindly emailing every pending order every five minutes.

`@Cron(CronExpression.EVERY_DAY_AT_2AM) handleDomainExpiryNotification()` is the daily domain expiry sweep. It fetches every domain that is anywhere near expiring from `DomainDetailBCRepoInterface.getExpiringDomains()`, then for each one checks whether its expiry timestamp falls inside tomorrow's 24 hour window (sending an "expiring tomorrow" email) or has already passed (sending an "already expired" email instead). Running this at 2 AM, a quiet traffic hour, rather than during the day, is a deliberate, sensible choice for a batch job that has to loop over every domain in the system one at a time.

`@Cron(CronExpression.EVERY_30_SECONDS) checkMintStatusJob()` is the fastest recurring job in the app, and it exists to poll a third party API (Freename) for the status of an on chain domain "mint" that was kicked off asynchronously earlier. It fetches every lifecycle record still marked pending, asks `freenameApiService.checkMintStatus(zoneName)` for that specific zone's current status, and once a mint comes back `COMPLETE` or `FAILED`, calls `freenameRegistrationService.finalizeFreenameMint` to close it out. Thirty seconds is fast because minting a domain on chain is not instantaneous, and the user experience of "your domain is being registered" only feels responsive if the backend notices the moment it finishes rather than making someone wait for the next slow batch job.

There is also a commented out, disabled duplicate of this exact same method sitting directly below the live one in the file, an earlier version of the same idea kept in place, presumably for reference, rather than deleted, worth noticing as another small example of how a long lived real codebase accumulates leftover history that nobody has cleaned up yet.

## `releaseReservedDomainsCron`, a method that is not actually a recurring cron job

`TasksService` also defines `releaseReservedDomainsCron()`, but notice it carries no `@Cron` decorator at all. It is a plain async method, and the only place it gets called is from `ReservedDomainSchedulerService` (`reserved-domain.scheduler.ts`), which is a genuinely different scheduling mechanism worth distinguishing clearly from everything above.

```ts
onModuleInit() {
    const targetDate = new Date('2026-02-19T10:00:00Z');
    const delay = targetDate.getTime() - Date.now();
    if (delay <= 0) { this.logger.warn('❌ Target time already passed, job not scheduled'); return; }
    const timeout = setTimeout(async () => {
        await this.customDomainServiceInterface.releaseReservedDomainsCron();
        this.schedulerRegistry.deleteTimeout('release-reserved-domains');
    }, delay);
    this.schedulerRegistry.addTimeout('release-reserved-domains', timeout);
}
```

This is a one time, hardcoded future timestamp, not a recurring interval. `NestJS`'s `SchedulerRegistry` is being used here for its `addTimeout`/`deleteTimeout` bookkeeping, which lets the job register itself by name and clean itself up after running exactly once, but the actual scheduling primitive underneath is a plain JavaScript `setTimeout`, calculated once when the app starts up, for one specific calendar moment. If that date has already passed by the time the app boots, the whole thing is skipped with a warning log rather than run immediately, which is the safe choice, a job meant for one specific release event should not fire retroactively just because the server happened to restart after that date. This is presumably a one off, time boxed operation, releasing a batch of domains that had been reserved ahead of some planned event, rather than an ongoing recurring task, and it is worth reading this file carefully specifically because it looks like a cron job (it lives in the `cron` folder, it calls a method literally named with "Cron" in it) without actually being one in the recurring sense every other job in this note is.

Once it does fire, `releaseReservedDomainsCron` calls `UDIntegrationService.releasedReserveDomainsOnUd` for every reserved domain, builds two Excel spreadsheets (successes and failures, via the `xlsx` package) summarizing the results, uploads both to S3 through `S3Service.uploadFile`, and emails the resulting download links to two named recipients, a fairly elaborate one shot reporting pipeline for what is ultimately a single scheduled event.

## The quick summary

This note originally counted six recurring jobs: unverified user email reminders every 8 hours, wallet balance checks every hour, pending order reminders every 5 minutes, domain expiry notifications daily at 2 AM, Freename mint status polling every 30 seconds, all inside `TasksService`, plus the AI alert threshold check every 6 hours inside `AiAlertService` from note 03. That count was already incomplete at the baseline commit, because other modules declare their own `@Cron` methods (listed in the update section below), and on the current `uat` branch the five `TasksService` jobs are not running at all because `CronModule` is commented out. Alongside those sits one genuinely one time, hardcoded date job (the reserved domain release) that only looks like a cron job because of where it lives and what it is named.

## Update from the October 2026 uat pull

This is the most operationally important change in the whole pull for this cluster, and it is a one line diff. Commit `836d5f89` ("Added new marketpalce v2 code", 10 September 2026) changed `src/app.module.ts` line 177 like this:

```ts
// src/app.module.ts (lines 174 to 180 on uat)
        StaticContentModule,
        AppraiselDomainModule,
        KeywordModule,
        // CronModule,
        DomainListingModule,
        TldsJsonModule,
        TransactionCheckCronModule,
```

The `import { CronModule } from '@components/cron/cron-module';` at line 10 is still there, which is why the build does not complain, but nothing else in `src` imports `CronModule`, `TasksService` or `ReservedDomainSchedulerService` (a search of the whole tree outside `src/components/cron/` finds only that unused import). In NestJS a provider is only constructed if some imported module lists it, and `@nestjs/schedule` only registers the `@Cron` methods of providers that were actually constructed. So with this line commented out, the following five recurring jobs from `src/components/cron/cron-service.ts` no longer run at all:

| Job | Schedule | Line | What stops happening |
|---|---|---|---|
| `handleCron` | every 8 hours | 73 | Unverified users stop receiving repeat verification emails (the up to five resends tracked by `sentEmailCount`). |
| `checkWalletBalance` | every hour | 91 | No low balance alert for the operational `WALLET_ADDRESS` (see the honest note below, it was already broken). |
| `handlePendingOrderReminderCron` | every 5 minutes | 118 | `DomainOrderService.sendPendingOrderReminderEmail()` is never called, so abandoned checkout reminder emails stop. |
| `handleDomainExpiryNotification` | daily at 2 AM | 130 | Users stop getting "expiring tomorrow" and "already expired" emails for domains in `tbl_domain_detail_bc`. |
| `checkMintStatusJob` | every 30 seconds | 335 | Pending Freename mints are never polled, so `finalizeFreenameMint` never runs automatically and those orders stay pending. |

The one shot `ReservedDomainSchedulerService` in `src/components/cron/reserved-domain.scheduler.ts` is also gone, but that one makes no practical difference, its hardcoded target of `2026-02-19T10:00:00Z` is already in the past, so it would only have logged "Target time already passed" and returned.

Of those five, the Freename one is the most serious, because it is the only automatic path that finishes a paid Freename registration. A grep for `finalizeFreenameMint` finds exactly two callers: this cron job, and `POST /internal/freename/finilize` in `src/components/freename/domain/freename-internal.controller.ts` lines 56 to 87. So on `uat` today the only way to finalise a Freename mint is for someone to call that internal route by hand, with the lifecycle record and status response in the body. That route has no `@UseGuards` at all, a security concern in its own right that becomes more tempting to rely on now that it is the only path. The expiry email job is the second most serious, because those emails are the trigger for renewal revenue, and nothing in the codebase replaces them.

The commit message does not explain why, but the code gives a strong hint. `TasksService`'s constructor (lines 58 to 71) throws `'QUICKNODE_RPC is not defined in environment variables'` if `INFURA_URL` is missing and throws again if `WALLET_ADDRESS` is missing, which would crash the whole application on boot in an environment, such as a developer's machine set up for marketplace v2 work, that does not have those secrets. Commenting out the module makes that crash go away. If that is the reason, the fix that keeps both worlds working is to guard the jobs with a feature flag or to read those values lazily inside each job, rather than switching every job off for every environment that runs this branch, and this line needs to be restored before `uat` is promoted to production. While you are in there, notice a real older bug that makes one row of the table moot anyway: `checkWalletBalance` uses `this.rpcUrl` at line 94, but the constructor stores the URL in a local `const rpcUrl` at line 59 and never assigns the class field declared at line 24, so `new Web3.providers.WebsocketProvider(undefined)` fails and the job has only ever logged "Error fetching wallet balance".

Everything outside `TasksService` is unaffected, because those jobs live in modules that are still imported. Reading every `@Cron` in `src` on `uat`, the jobs that still run are `AiAlertService` every 6 hours (`src/components/ai/ai-alert.service.ts` line 60), the blog `pruneOldViewEvents` at 2 AM UTC (`src/components/blog/prune.service.ts` line 22), the GA4 sync at 3 AM (`src/components/google-analytics/services/ga4-sync-cron.service.ts` line 20), the four v1 marketplace jobs in `src/components/marketplace/transaction-cron/cron.service.ts` (`checkBlockchainTransactions`, `checkBuyDomainTransactions` and `markExpiredDomains` every minute at lines 123, 332 and 538, and `updateTrendingScores` every hour at line 575), the reputation recalculation at 3 AM (`src/components/reputation-gm-perk/reputation/services/reputation.service.ts` line 408, the one that sets the `getCronRunning` freeze flag used by deployments and perks), the Search Console sync at 4 AM (`src/components/search-console/services/gsc-sync-cron.service.ts` line 27), and the search log rollup every 15 minutes (`src/components/search-management/search-log-rollup.service.ts` line 23). All of those already existed at the baseline commit, this note simply had not listed them. The pull also adds four new six field cron expressions in marketplace v2's `src/components/marketplacev2/market-data/market-data.scheduler.ts` lines 61 to 76, driven by `MARKET_CRON` in `market-data.constants.ts` lines 72 to 77 (slugs daily at 03:00:00, stats at seconds 0 of minutes 0, 15, 30 and 45, listings at minute 7 of every hour, activity every minute), covered in [../12-marketplace-v2/10-opensea-market-data-pipeline.md](../12-marketplace-v2/10-opensea-market-data-pipeline.md), and the on chain poller, which runs on its own `setInterval` tick loop (`src/components/marketplacev2/poller/tick/chain-event-source-poller.service.ts` line 120) rather than `@Cron`, covered in [../12-marketplace-v2/08-the-onchain-event-poller-architecture.md](../12-marketplace-v2/08-the-onchain-event-poller-architecture.md).

One small leftover for completeness: `ScheduleModule.forRoot()` is called three times in the app, in `src/app.module.ts` line 121, `src/components/auth/auth.module.ts` line 22 and `src/components/listener/listener.module.ts` line 9. NestJS tolerates this, but only the root one is needed. The existing `src/components/cron/cron-service.spec.ts` was not changed in this pull and is unaffected by the commented out import, because it constructs `TasksService` directly with `new TasksService(...)` at line 42 rather than through `AppModule`, which is a good reminder that a green unit test cannot tell you whether a module is actually wired into the running app.
