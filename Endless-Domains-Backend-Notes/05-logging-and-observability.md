# 05. Logging and Observability

## Why a real production app needs more than `console.log`

Every reference project you have read before this one used Nest's built in `Logger` or plain `console.log`, fine for a learning project running on one machine you can watch directly. Endless Domains runs on real infrastructure, likely containers or servers you cannot casually attach a terminal to, so its logs need to travel somewhere durable and searchable, in this case, AWS CloudWatch. This is one of the more directly useful things to absorb from this codebase if you are aiming at fullstack or backend roles, structured, shippable logging is a genuinely different skill from printing helpful messages to your own terminal.

## `CustomLoggerService`

```ts
// src/logger/file-logge.ts
this.logger = winston.createLogger({
    levels: winston.config.npm.levels,
    transports: [
        new CloudWatchTransport({ logGroupName: secrets.LOG_GROUP_NAME, ..., level: 'error' }),
        new CloudWatchTransport({ logGroupName: secrets.LOG_LOGGER_GROUP_NAME, ..., level: 'info', format: this.filterOnlyLevel('info') }),
    ],
});
```

This wraps `winston`, a widely used Node.js logging library, configured with two separate CloudWatch destinations, one log group that only receives error level entries, and a second, separate log group that receives info level entries (filtered so this second stream genuinely only carries info, not everything at or below it). Splitting errors and informational events into two separate destinations like this is a deliberate operational choice, it means someone monitoring production can watch the error log group specifically and get paged or alerted only on real problems, without that signal being buried in routine informational noise, while the info stream stays available separately for debugging or auditing when needed.

## Typed, structured log methods

Beyond the plain `log`, `error`, `warn`, and `debug` methods every `Logger` has, `CustomLoggerService` also defines specific, typed methods, `logWebhookEvent`, `logOrderEvent`, `logDomainEvent`, and `logOrderCancelled`, each accepting a strongly typed payload object (defined in `logger.types.ts`, worth opening directly to see the exact shape of `WebhookLogPayload`, `OrderLogPayload`, `DomainLogPayload`, and the `LogCategory` and `LogEvent` enums) rather than a loose, freeform string. Every one of these builds a consistent `StructuredLogEntry` object, a timestamp, a category, a correlation id, an order id and number where relevant, a domain name where relevant, an event name, a human message, and a free form detail object, before handing it to Winston.

This matters more than it might look like at first. A plain string log like `"order 4821 failed"` is only useful to a human reading it in the moment. A structured entry like this one can be queried later, show me every `ORDER_CANCELLED` event for `correlationId X`, or count how many `WEBHOOK_STRIPE` events happened today, because CloudWatch (or whatever tool reads these logs) can filter and aggregate on real fields, not just search raw text. The `correlationId`, built with `buildCorrelationId` from `logger.types.ts`, is the mechanism that lets you trace one single request or one single business event (a domain purchase, say) across multiple, separately logged steps, an order being created, a webhook confirming payment, a domain being provisioned, by giving all of them the same id, this is the same underlying idea as the request id you may have seen a frontend network tab display for tracing a single API call, just extended across an entire backend, asynchronous, multi step process.

## Where this actually gets used

`LoggerModule` exports `CustomLoggerService` for any feature module to inject, and the specific `logOrderEvent`, `logDomainEvent`, and `logWebhookEvent` calls are worth watching for once you reach the commerce, domain, and webhook cluster notes, they mark the exact points in the code someone already decided were important enough to leave a durable, structured trail for, which is often a faster way to find the truly critical logic in a large, unfamiliar codebase than reading every file start to finish, follow the logging calls, and you will usually land on the code that actually matters most to the business.

## Update from the October 2026 uat pull

One small change landed in `CustomLoggerService` between `a131b429` and `dc1ba3e8`, and it matters far more than its size suggests, because the new marketplace leans on `warn()` heavily.

```ts
// src/logger/file-logge.ts
    warn(message: string) {
        this.logger.warn({ context: this.context, message });
        console.log(new Date().toISOString(), this.context, message, 'warn')
    }
```

`warn()` now also prints to the console, which makes warnings visible in a local terminal and in the container's stdout. The history behind that line is worth knowing. A CloudWatch transport for the warn level was added in commit `96f221ec` and removed again in `bdd8e80a`. So on `uat` today, `warn` messages reach the console only. They go to neither the error log group (which keeps only `error`) nor the info log group (which filters to exactly `info`). `debug()` goes nowhere at all outside the console.

That matters because a lot of important marketplace v2 signals are logged at `warn` or `debug`. These include a failed `getLogs` call in the poller, the alert that fires when an order the backend believed was cancelled was sold on chain anyway, Seaport `CounterIncremented` events (which silently invalidate a seller's orders), every fallback warning from the chain config loader (which uses `console.warn` directly), and the many "missing contract key" warnings from the market data config. Unless someone is reading raw container output, none of these will be seen in CloudWatch. If you are ever asked why an alert never fired, check the log level first.

Two smaller observations for the curious. `filterOnlyLevel` near the top of the file is declared `async`, so it returns a Promise to Winston's `format` rather than the log entry itself, which is worth testing before you trust that filter. And the logger sets up its CloudWatch transports asynchronously, while the new poller calls `debug()` from its constructor during boot, which is a latent race the poller notes describe in [clusters/12-marketplace-v2/08-the-onchain-event-poller-architecture.md](clusters/12-marketplace-v2/08-the-onchain-event-poller-architecture.md).
