# 02. High Level Architecture and Bootstrap

## Why this note exists before any feature module does

Every teaching project you have notes on had one `main.ts` and one `app.module.ts` that were simple enough to read in thirty seconds. This one is not, `app.module.ts` alone imports roughly ninety separate feature modules, and `main.ts` does real production hardening work that none of the smaller projects needed. Understanding these two files well is what lets everything else in this codebase make sense, because every single feature module, no matter which cluster of notes covers it, gets wired into the exact same request pipeline described here.

## `main.ts`, read top to bottom

```ts
const app = await NestFactory.create(AppModule);
useContainer(app.select(AppModule), { fallbackOnErrors: true });
```

The first line is the same one you have already seen build every Nest app in every other project, it walks `AppModule`'s import tree and constructs the whole dependency injection container. The second line is new, `useContainer` from `class-validator` tells `class-validator` to resolve its own dependency injected constraints (custom validators that themselves need a service injected, covered in the `@core` note) through Nest's container rather than instantiating them blindly, `fallbackOnErrors: true` means if that resolution fails for some validator, it falls back to a plain `new` instead of crashing the whole app.

```ts
const allowedOrigins = [ "http://localhost:3000", ... "https://endlessdomains.io", ... ];
app.enableCors({
    origin: (origin, callback) => {
        if (!origin) return callback(null, true);
        if (allowedOrigins.includes(origin)) { callback(null, true); }
        else { console.warn(...); callback(new Error(...)); }
    },
    credentials: true,
    methods: "GET,HEAD,PUT,PATCH,POST,DELETE,OPTIONS",
    allowedHeaders: "Content-Type, Authorization, X-Requested-With",
});
```

This is a real, production grade CORS policy, worth comparing against the plain `app.enableCors()` (allow everything) you may have seen in smaller projects, or even against the commented out `// app.enableCors();` sitting directly above it, a trace of an earlier, looser version of this same line that a past engineer replaced with this explicit allowlist. Every real domain this company owns, its staging environments, and its local development ports are named explicitly, and everything else is rejected with a logged warning. `credentials: true` is what allows cookies (used for the refresh token flow, covered in the auth cluster notes) to travel across these origins at all, browsers refuse to send cookies cross origin unless a server explicitly opts in like this.

```ts
app.setGlobalPrefix('api/v1');
```

Every single route in every one of the roughly ninety feature modules is automatically prefixed with `/api/v1`, this one line is why a controller decorated with `@Controller('domain')` actually answers at `/api/v1/domain`, not `/domain`. This is also a versioning signal, worth noticing for later, a `v2` prefix existing alongside this one someday would be how this team would introduce a breaking API change without breaking every existing frontend client at once.

```ts
app.use(bodyParser.json({ limit: '50mb', verify: (req, _res, buf) => { req.rawBody = buf; } }));
app.use(bodyParser.urlencoded({ limit: '50mb', extended: true }));
```

A 50mb request body limit is unusually large for a typical JSON API, worth keeping in mind when you reach the modules that justify it (bulk CSV or Excel uploads, and large webhook payloads). The `verify` callback capturing `req.rawBody` is a detail that matters a great deal later, several webhook providers (Stripe among them, see the payment gateway integration cluster notes) cryptographically sign their webhook payload using its exact raw bytes, and if you only have the already parsed JSON object, you cannot verify that signature, this line exists specifically so a webhook controller elsewhere in the app can still get at the original, unparsed bytes.

```ts
app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }));
app.use(helmet());
app.use(cookieParser());
```

The same global `ValidationPipe` pattern from the smaller reference projects, `whitelist: true` strips any property off an incoming request body that is not declared on the target DTO, `transform: true` is what lets a route handler receive a real, typed class instance instead of a plain object, and is also why numeric string route params can be automatically converted to real numbers when a DTO or pipe expects one. `helmet()` sets a batch of security related HTTP response headers in one line, and `cookieParser()` is what lets a controller read cookies off `req.cookies`, needed for the refresh token flow.

```ts
const secretsService = new SecretsService();
const secrets = await secretsService.getSecret(process.env.AWS_MANAGER);
```

This is the single most important line to understand before touching anything else in this codebase, and it is covered in full in [03-configuration-and-secrets.md](03-configuration-and-secrets.md), this app's real configuration values do not live in a `.env` file the way every smaller reference project's did, they live in AWS Secrets Manager, fetched once at startup using one environment variable, `AWS_MANAGER`, that names which secret bundle to fetch.

```ts
const apiCallLimiter = rateLimit({ windowMs: ..., max: ..., message: {...} });
app.use('/api/v1/auth/forgot-password', apiCallLimiter);
app.use('/api/v1/domain/detail/refresh_domain', apiCallLimiter);
```

This is a second, independent rate limiting mechanism, using the `express-rate-limit` package directly, applied only to these two specific routes, sitting alongside the separate `ThrottlerModule` you will see registered in `app.module.ts`, which, as the update section below explains, only protects the handful of routes that opt into it explicitly. Two specific, sensitive routes (three since the October 2026 update), forgot password (an easy target for abuse, since triggering it repeatedly floods a real inbox) and a domain detail refresh endpoint, get an extra, tighter, independently configured limit on top of the global one, this is a deliberate defense in depth choice worth remembering when you reach the throttler note.

```ts
config.update({ accessKeyId: ..., secretAccessKey: ..., region: ... }); // legacy aws-sdk v2
await Moralis.start({ apiKey: secrets.MORALIS_API_KEY });
const PORT = secrets.PORT || 3000;
await app.listen(PORT);
```

The AWS credentials fetched from Secrets Manager get pushed into the older, global `aws-sdk` (v2) configuration object, needed because some parts of this codebase still use that older SDK style alongside the newer, modular `@aws-sdk/client-*` packages you will see used elsewhere, a sign of a codebase that has been incrementally modernized rather than rewritten wholesale, completely normal in a real, multi year production system. `Moralis.start(...)` initializes a blockchain data API client once, globally, at boot, rather than per request, exactly the same "connect once at startup" instinct behind `PrismaService.onModuleInit` in the smaller reference projects. The port itself even comes from the fetched secrets bundle rather than a plain environment variable.

## `app.module.ts`, the shape rather than every line

Reading all roughly ninety import lines one at a time would not teach you anything a summary cannot, what is worth understanding is the shape. Three top level, infrastructure style modules configure themselves once, `ConfigModule.forRoot` (loading its data from `SecretsService`, again see the config note), `ThrottlerModule.forRoot([{ ttl: 60000, limit: 60 }])` (which looks like a global rate limit of sixty requests per minute per client, but is not one, see the correction in the update section at the bottom of this note), and `TypeOrmModule.forRootAsync({ useClass: TypeOrmConfigService })` (the database connection, covered in the next note). `CacheModule.register({ isGlobal: true, ttl: 60000 })` sets up a short lived, in memory cache, and the comment directly above it in the real file is worth reading verbatim, it explains this exists specifically for read heavy, slow changing dashboard and analytics endpoints that use a `CacheInterceptor`, not for every route. `ScheduleModule.forRoot()` and `EventEmitterModule.forRoot()` turn on, application wide, the ability for any module to declare a cron job or emit and listen for internal application events, both covered in their own cluster notes.

Every remaining import is a feature module, one Nest module per business capability, and the sheer number of them, not any single one of them, is the actual architectural lesson here. This is what a real, several years old production system built with the same `@Module({ imports, controllers, providers })` pattern you already know actually grows into, dozens of small, mostly self contained feature modules, each with its own controller, service, and entities, all wired together at the very top by one `AppModule` that does nothing but list them. Nothing about the underlying pattern changed from the smallest reference project you have already read, only the number of times it gets repeated.

## Where to go next

[03-configuration-and-secrets.md](03-configuration-and-secrets.md) covers exactly how a value gets from AWS into a running piece of code. [04-database-typeorm-and-repositories.md](04-database-typeorm-and-repositories.md) covers how every one of those roughly ninety modules actually talks to Postgres. After that, the `clusters` folder groups the feature modules themselves by business area.

## Update from the October 2026 uat pull

This note was first written against commit `a131b429`. Since then, up to `uat` merge commit `dc1ba3e8`, both `main.ts` and `app.module.ts` have changed in small but important ways, and one statement above turned out to be wrong even at the time it was written. This section covers all of it.

### The stack is newer than CLAUDE.md says

Open `package.json` and you will find `"@nestjs/core": "^10.4.22"`, `"typeorm": "^0.3.17"`, `"@nestjs/cache-manager": "^3.1.3"` with `"cache-manager": "^7.2.9"`, and, new in this update, `"ethers": "^6.13.4"` (commits `71fd2fec` and `26b1f0e4`, up from `^5.7.2`). The project `CLAUDE.md` still lists NestJS 8, TypeORM 0.2 and ethers 5. When a document and the code disagree, the code wins, so check `package.json` yourself before trusting any version table, including the one in CLAUDE.md. Two other dependencies arrived with the new marketplace. The first is `@endlessdomains/order-builder`, a private package installed straight from the company's `ed-shared-package` GitHub repository and pinned to commit `855e344a`, so a fresh machine or CI runner needs GitHub credentials to install it (and `package.json` says `git+https` while `package-lock.json` says `git+ssh`, which can break `npm ci` on a build host with no SSH key). The second is `an-array-of-english-words`, used to classify "dictionary word" domains. Both are explained in [clusters/12-marketplace-v2/01-what-marketplace-v2-is-and-the-module-map.md](clusters/12-marketplace-v2/01-what-marketplace-v2-is-and-the-module-map.md).

### `main.ts`: one more route behind the shared limiter

```ts
// src/main.ts
app.use('/api/v1/auth/forgot-password', apiCallLimiter);
app.use('/api/v1/domain/detail/refresh_domain', apiCallLimiter);
app.use('/api/v1/marketplacev2/orders/listing-status/sync', apiCallLimiter);
```

The new third line protects the marketplace v2 listing status sync endpoint, which triggers a full refresh of the user's domains from Alchemy and Moralis and is therefore expensive to call repeatedly. There is a subtlety here worth understanding, because it is a classic mistake. All three lines pass the same `apiCallLimiter` instance, and `express-rate-limit` keeps one counter per client IP per instance, not per route. A user who calls forgot password a few times has therefore used up part of their budget for the domain refresh and the listing sync too, because the three routes share a single per IP allowance. On top of that, the limiter's default memory store lives inside each Node process, so with several PM2 processes or several ECS tasks running, each one counts separately and the real limit is multiplied by the number of processes. If you ever want three independent limits, the fix is three separate `rateLimit(...)` calls, and if you want a limit that holds across processes, you need a shared store such as Redis.

Something is still missing from `main.ts` that the new background loops would benefit from. There is no `app.enableShutdownHooks()`, so the `onModuleDestroy` hooks never run on SIGTERM and the process is simply killed. The poller and the listing expiry scheduler both define one of these hooks to stop their timers. That is harmless today, but worth knowing when you write your first background worker.

### `app.module.ts`: one module added, one module switched off

```ts
// src/app.module.ts
import { Marketplacev2Module } from '@components/marketplacev2/marketplacev2.module';
// ...
        KeywordModule,
        // CronModule,
        DomainListingModule,
// ...
        ThirdPartyReputationModule,
        Marketplacev2Module,
```

`Marketplacev2Module` is a barrel module that pulls in the wallet verification, order, listing status, transaction, poller and market data submodules, all covered in [cluster 12](clusters/12-marketplace-v2/00-README.md). Two of its submodules behave differently at boot from every other feature module in the app. The chain config module loads and validates its keys from the AWS secret during startup and throws if any required one is missing or malformed. The poller checks at construction time that its hardcoded event topic hashes match the ABI. Either failure stops the whole API from booting, not just the marketplace, because Nest treats any provider that throws during initialization as fatal for the entire application. That is a deliberate fail fast choice, since a misconfigured marketplace that silently accepted orders against the wrong contract would be far worse. It does mean a typo in one marketplace secret takes down login, search and checkout too.

`CronModule` is now commented out (commit `836d5f89`), yet it is still imported, unused, near the top of the file. This is easy to miss and has real consequences. The legacy `TasksService` jobs in `src/components/cron/cron-service.ts` no longer run on `uat` at all. Those jobs resent verification emails every eight hours, sent pending order reminders every five minutes, sent the daily domain expiry emails, finalized Freename mints every thirty seconds, and ran an hourly wallet balance check. The most likely reason it was switched off is that the `TasksService` constructor throws at boot when `INFURA_URL` or `WALLET_ADDRESS` is missing. The full list and its consequences are in [clusters/10-ai-and-infra-utilities/05-cron-scheduled-jobs.md](clusters/10-ai-and-infra-utilities/05-cron-scheduled-jobs.md). Every other `@Cron` in the codebase, including the four new market data jobs, still runs, because `ScheduleModule.forRoot()` is still registered.

### A correction: `ThrottlerModule` is not a global rate limit

The original version of this note described `ThrottlerModule.forRoot([{ ttl: 60000, limit: 60 }])` as a global limit of sixty requests per minute sitting underneath everything. That was wrong, and it is a mistake worth learning from because it is so easy to make. In `@nestjs/throttler`, `forRoot` only configures the limits. Nothing is actually throttled unless `ThrottlerGuard` runs, either globally through a provider like `{ provide: APP_GUARD, useClass: ThrottlerGuard }` or per route through `@UseGuards(ThrottlerGuard)`. A search of `src` finds no `APP_GUARD` anywhere. It finds `ThrottlerGuard` applied in only a handful of places: the blog view counter (`blog/views.controller.ts:26`), one admin analytics route (`google-analytics/controllers/admin-analytics.controller.ts:108`), the waitlist (`waitlist/waitlist.controller.ts:52`), and an import in `auth.controller.ts`. Every other route has no throttling beyond the three `express-rate-limit` paths in `main.ts`, and that includes all the new public marketplace v2 and `/market` endpoints. The lesson applies well beyond this file: registering a module is not the same as applying it, and the only way to know whether a protection is really in force is to find the line that applies it.

### The in memory cache now carries real state

`CacheModule.register({ isGlobal: true, ttl: 60000 })` was originally described as a small cache for slow dashboard endpoints. Marketplace v2 now uses the same in process store for much more important things. It holds the wallet proof nonces (`marketplacev2:prove-wallet-nonce:<userId>`, with a five minute TTL) and every OpenSea market data payload (`market:stats`, `market:series`, `market:activity` and three more, all written with no expiry). Because this store lives inside one Node process, it is empty after every restart or deploy, so the market endpoints return 503 until the first jobs finish. It is also not shared between processes, so a nonce issued by one ECS task is unknown to another. Both points are explained in detail in cluster 12, notes 02 and 10.

### Two response conventions now

Everything in this app used to follow one shape: `/api/v1/<feature>` returning the `Response` envelope `{ success, statusCode, message, result }`. Marketplace v2 mostly follows it under `/api/v1/marketplacev2/...`, but its public market data controller is mounted at `/api/v1/market/stats`, `/series` and `/activity` and returns raw DTOs with no envelope. A frontend with a single fetch helper that always reads `.result` needs a special case for those three routes.
