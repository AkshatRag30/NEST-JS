# Phase 1 PRD: Foundations and Config

## Goal

Stand up the empty shell of the application with every cross cutting concern already wired in correctly, before a single feature module exists. Every later phase depends on this one being solid, a mistake here shows up everywhere, not just once.

## Concepts practiced

Project setup and the Nest CLI, modules, `ConfigModule`, environment variable validation, `PrismaService` and `PrismaModule`, `MongooseModule.forRoot`, lifecycle hooks (`OnModuleInit`), a global logging middleware, a global exception filter, and a global `ValidationPipe`. See the relevant rows in [03-concept-coverage-map.md](03-concept-coverage-map.md).

## Scope

Generate a fresh Nest project. Install `@nestjs/config`, `@prisma/client` and `prisma`, `@nestjs/mongoose` and `mongoose`, `class-validator` and `class-transformer`. Set up `ConfigModule.forRoot({ isGlobal: true })`, unlike the reference MongoDB project, which called `forRoot()` without the global flag and then never actually used `ConfigService` anywhere, here it must be marked global, and `ConfigService` must genuinely be the thing every later module reads its settings from, not raw `process.env` reads scattered around.

Write a small env validation step, either with a Joi schema passed to `ConfigModule.forRoot({ validate: ... })` or a manually written function that checks for `DATABASE_URL`, `MONGO_URI`, and `JWT_SECRET` and throws a clear, readable error naming exactly which variable is missing if any of them are absent. This single step is the direct fix for the failure mode found in every one of the eight reference projects, a missing `MONGO_URI` or `DATABASE_URL` that either crashes with a confusing low level driver error or, worse, never gets checked at all until some unrelated request happens to fail later.

Write `PrismaService`, extending `PrismaClient`, implementing `OnModuleInit` to call `this.$connect()` explicitly and log a clear one line success message, and wrap it in a `PrismaModule` decorated `@Global()`, so that unlike the reference Prisma project, every later feature module can inject `PrismaService` without importing `PrismaModule` itself. Connect to MongoDB with `MongooseModule.forRoot(configService.get('MONGO_URI'))`, using the async `forRootAsync` form specifically so the connection genuinely goes through `ConfigService` rather than a raw `process.env` read sitting inside `AppModule`, closing the same gap found in the MongoDB reference project.

Write one logging middleware, applied globally in `main.ts` with `app.use(...)`, that logs the HTTP method, path, and response time of every request. Write one global exception filter, applied with `app.useGlobalFilters(...)`, that catches both Nest's built in `HttpException` family and any unexpected error, and always responds with one consistent JSON shape containing a status code, a message, and a timestamp. Register the global `ValidationPipe` with `whitelist: true` and `forbidNonWhitelisted: true`, so that every DTO written from this point forward is actually enforced, not just decorated for show the way the inert GraphQL input validators were in the reference project.

## Acceptance criteria

1. Starting the app with any one of the three required environment variables missing produces one clear startup error naming that exact variable, and the app does not attempt to serve any requests.
2. Starting the app with all three variables present logs one line confirming the Postgres connection and one line confirming the MongoDB connection before the app reports it is listening.
3. Sending a request with a body that fails a DTO's validation rules, once the first real DTO exists in phase 2, returns a 400 with a readable list of which fields failed and why, not a raw stack trace.
4. Every response, success or failure, is visible in the server log with its method, path, and timing.
5. An unexpected thrown error anywhere in the app, simulated by temporarily throwing a plain `Error` inside a placeholder route, still comes back as the same consistent JSON error shape, not an unhandled exception.

## Explicit trap to avoid

Do not let `AppModule` read any environment variable directly with `process.env`. Every read must go through `ConfigService`, and the validation step must run before any module that depends on those variables gets a chance to initialize. This is the single most repeated mistake across the eight reference projects, and fixing it here means it never has a chance to repeat itself later.
