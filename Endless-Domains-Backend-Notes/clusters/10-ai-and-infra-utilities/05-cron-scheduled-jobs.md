# 05. Cron: what actually runs on a schedule, and how often

## The ScheduleModule, and where its jobs actually live

`app.module.ts` registers `ScheduleModule.forRoot()` once, near the top of its imports, which is what turns on NestJS's own `@Cron` decorator support application wide. That one registration is what lets any injectable class anywhere in this codebase declare a method that runs on a schedule, simply by decorating it, no further wiring needed per job. Because of that, the actual scheduled jobs are not all sitting in one obvious place, `CronModule` is the dedicated home for most of them, but `AiAlertService` from note 03 declares its own `@Cron` method too, entirely separately, since any provider anywhere in the module graph can do the same thing.

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

Six real, recurring jobs exist in this codebase today: unverified user email reminders every 8 hours, wallet balance checks every hour, pending order reminders every 5 minutes, domain expiry notifications daily at 2 AM, Freename mint status polling every 30 seconds, all inside `TasksService`, plus the AI alert threshold check every 6 hours inside `AiAlertService` from note 03. Alongside those sits one genuinely one time, hardcoded-date job (the reserved domain release) that only looks like a cron job because of where it lives and what it is named.
