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

This is a second, independent rate limiting mechanism, using the `express-rate-limit` package directly, applied only to these two specific routes, sitting alongside the separate, application wide `ThrottlerModule` you will see registered in `app.module.ts`. Two specific, sensitive routes, forgot password (an easy target for abuse, since triggering it repeatedly floods a real inbox) and a domain detail refresh endpoint, get an extra, tighter, independently configured limit on top of the global one, this is a deliberate defense in depth choice worth remembering when you reach the throttler note.

```ts
config.update({ accessKeyId: ..., secretAccessKey: ..., region: ... }); // legacy aws-sdk v2
await Moralis.start({ apiKey: secrets.MORALIS_API_KEY });
const PORT = secrets.PORT || 3000;
await app.listen(PORT);
```

The AWS credentials fetched from Secrets Manager get pushed into the older, global `aws-sdk` (v2) configuration object, needed because some parts of this codebase still use that older SDK style alongside the newer, modular `@aws-sdk/client-*` packages you will see used elsewhere, a sign of a codebase that has been incrementally modernized rather than rewritten wholesale, completely normal in a real, multi year production system. `Moralis.start(...)` initializes a blockchain data API client once, globally, at boot, rather than per request, exactly the same "connect once at startup" instinct behind `PrismaService.onModuleInit` in the smaller reference projects. The port itself even comes from the fetched secrets bundle rather than a plain environment variable.

## `app.module.ts`, the shape rather than every line

Reading all roughly ninety import lines one at a time would not teach you anything a summary cannot, what is worth understanding is the shape. Three top level, infrastructure style modules configure themselves once, `ConfigModule.forRoot` (loading its data from `SecretsService`, again see the config note), `ThrottlerModule.forRoot([{ ttl: 60000, limit: 60 }])` (a global rate limit of sixty requests per minute per client, sitting underneath the two specific, tighter limits from `main.ts`), and `TypeOrmModule.forRootAsync({ useClass: TypeOrmConfigService })` (the database connection, covered in the next note). `CacheModule.register({ isGlobal: true, ttl: 60000 })` sets up a short lived, in memory cache, and the comment directly above it in the real file is worth reading verbatim, it explains this exists specifically for read heavy, slow changing dashboard and analytics endpoints that use a `CacheInterceptor`, not for every route. `ScheduleModule.forRoot()` and `EventEmitterModule.forRoot()` turn on, application wide, the ability for any module to declare a cron job or emit and listen for internal application events, both covered in their own cluster notes.

Every remaining import is a feature module, one Nest module per business capability, and the sheer number of them, not any single one of them, is the actual architectural lesson here. This is what a real, several years old production system built with the same `@Module({ imports, controllers, providers })` pattern you already know actually grows into, dozens of small, mostly self contained feature modules, each with its own controller, service, and entities, all wired together at the very top by one `AppModule` that does nothing but list them. Nothing about the underlying pattern changed from the smallest reference project you have already read, only the number of times it gets repeated.

## Where to go next

[03-configuration-and-secrets.md](03-configuration-and-secrets.md) covers exactly how a value gets from AWS into a running piece of code. [04-database-typeorm-and-repositories.md](04-database-typeorm-and-repositories.md) covers how every one of those roughly ninety modules actually talks to Postgres. After that, the `clusters` folder groups the feature modules themselves by business area.
