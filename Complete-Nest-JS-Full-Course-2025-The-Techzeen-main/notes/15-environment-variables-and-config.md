# 15. Environment Variables and Config

## What an environment variable is, and why hardcoding values is a problem

An environment variable is a named value that lives outside your code, in the environment the process is running in, rather than being written directly into a source file. The slide deck's definition: "used to store values that change depending on the environment (development, production, testing), like database URLs, API keys, secrets, etc."

Imagine you hardcoded your real database password directly into a `.ts` file: `const dbPassword = "sup3rS3cret";`. Two problems immediately follow. First, if that file is ever committed to a public GitHub repository (a shockingly common real-world mistake), your production database password is now public. Second, the moment you need to run this same code against a different database, for testing, for a staging environment, for a different developer's local machine, you would have to go edit the source code itself and hope nobody accidentally commits their personal value over the real one. Environment variables solve both problems: the value lives outside the code entirely, is different per machine/environment, and never needs to be committed to version control at all.

## Why `.env` files are gitignored

Open `.gitignore` in the root of this project and you will find:

```
.env
.env.development.local
.env.test.local
.env.production.local
.env.local
```

This is why, if you clone this repository, there is no `.env` file sitting in it. `.env` files are meant to be created locally by each developer (or generated during deployment), containing that specific environment's actual secret values, and they are deliberately excluded from git so secrets never get committed to source control. In a real team, you would typically find an `.env.example` file (which is safe to commit, since it only lists variable names with placeholder or blank values) showing new developers exactly which variables they need to define locally.

## Wiring up `@nestjs/config`

This project uses the official `@nestjs/config` package (listed in `package.json`'s dependencies) to read environment variables cleanly, instead of reaching directly for Node's raw `process.env` everywhere. It is registered in `src/app.module.ts`:

```ts
imports: [EmployeeModule, CategoryModule, StudentModule, CustomerModule, ConfigModule.forRoot({
  isGlobal: true,
})],
```

`ConfigModule.forRoot({...})` sets up the configuration system for the whole app. By default, it automatically looks for a `.env` file in your project's root directory and loads every key/value pair inside it into a form your app can read through `ConfigService`.

`isGlobal: true` is the option worth understanding precisely. Normally in NestJS, if `CustomerModule` wanted to use something exported by `ConfigModule`, it would need to explicitly `import` `ConfigModule` itself, inside `customer.module.ts`, following the exact same import chain rules explained in [04-modules.md](04-modules.md). `isGlobal: true` skips that requirement entirely: once `ConfigModule` is registered as global anywhere (here, in the root `AppModule`), its exported `ConfigService` becomes available for injection into any provider, anywhere in the app, without that module needing to import `ConfigModule` itself.

## Reading a variable with `ConfigService`

`src/ev/ev.service.ts`:

```ts
import { Injectable } from '@nestjs/common';
import {ConfigService} from '@nestjs/config';
@Injectable()
export class EvService {
    constructor(private configureService: ConfigService){}

    getDbUrl(){
        return this.configureService.get<string>('DATABASE_URL')
    }
}
```

`ConfigService` is injected through the constructor exactly the same way `CategoryService` was injected into `CategoryController` (see [07-dependency-injection.md](07-dependency-injection.md)), because thanks to `isGlobal: true`, NestJS can supply it here without `EvService`'s own module needing to import `ConfigModule` directly.

`this.configureService.get<string>('DATABASE_URL')` reads the environment variable named `DATABASE_URL`. The `<string>` is a TypeScript generic telling `ConfigService` (and your editor) what type to expect back, purely for type safety on your end, it does not change what value is actually returned. If a `.env` file existed in this project's root containing a line like:

```
DATABASE_URL=mongodb://localhost:27017/mycourse
```

then `getDbUrl()` would return that exact string. If no `.env` file exists at all, or that specific key is missing from it, `get()` returns `undefined` instead (you can also pass a default value as a second argument, `get('DATABASE_URL', 'some-fallback-value')`, though this project's code does not do that here).

This is exposed through `src/ev/ev.controller.ts`:

```ts
@Controller('ev')
export class EvController {
    constructor(private readonly evService: EvService){}
    @Get()
    getUrl(){
        return this.evService.getDbUrl();
    }
}
```

`GET /ev` returns whatever `DATABASE_URL` currently resolves to. Since this repo has no `.env` file committed (by design, as explained above), running this project fresh and hitting this route would return an empty response, because the variable is genuinely undefined. To actually see this work, you would create a `.env` file yourself in the project root and add a `DATABASE_URL=...` line to it.

## `process.env` directly: the other place you have already seen this

Environment variables do not always need `@nestjs/config`. You already saw the plain Node.js way in `main.ts`:

```ts
await app.listen(process.env.PORT ?? 3000);
```

`process.env` is a plain JavaScript object (provided by Node.js itself, no package required) containing every environment variable currently set for this process. `process.env.PORT` reads a variable named `PORT` directly. `@nestjs/config`'s `ConfigService` is essentially a more structured, dependency-injection-friendly wrapper around this same underlying mechanism, plus the convenience of automatically loading a `.env` file into `process.env` for you at startup. Both approaches read from the exact same underlying source.

## Why this is genuinely important beyond just "keeping secrets"

The slide deck's "Why We Use Them" points map directly onto real workflows: securing sensitive info means your database password or a third party API key never appears in your git history at all. Easily switching between dev/staging/prod environments means the exact same compiled code can run against a completely different database or configuration just by changing which `.env` values are present on that machine, with zero code changes. Keeping the codebase clean and configurable means a value like a port number, or a feature flag, does not need a code change and a redeploy just to be tweaked.

## Key terms recap

Environment variable: a configuration value supplied outside the source code, through the operating environment the process runs in.

`.env` file: a plain text file of `KEY=value` pairs, typically excluded from version control, used to supply environment variables during local development.

`ConfigModule.forRoot({ isGlobal: true })`: sets up `@nestjs/config`, loading `.env` values and making `ConfigService` injectable anywhere in the app without needing to re-import `ConfigModule` in every module.

`ConfigService.get<T>('KEY')`: reads a specific environment variable in a type-safe way.

`process.env`: Node.js's own raw object holding every currently set environment variable, which `@nestjs/config` builds on top of.
