# 01. Project Setup and Bootstrap

## What this project actually is

This repository is a minimal NestJS application whose entire purpose is to show how to bolt rate limiting onto an API. It was generated from the standard Nest CLI starter template and then trimmed down to exactly four source files, `app.module.ts`, `app.controller.ts`, `app.service.ts`, and `main.ts`, plus one unit test and one end to end test. There is no database, no extra feature modules, and no DTOs anywhere in this project. Everything you need to understand it lives in those handful of files, which is exactly why these notes can stay short and still be complete.

## `package.json`, the project's identity and scripts

```json
{
  "name": "rate-limit",
  "version": "0.0.1",
  "description": "",
  "author": "",
  "private": true,
  "license": "UNLICENSED",
  "scripts": {
    "build": "nest build",
    "format": "prettier --write \"src/**/*.ts\" \"test/**/*.ts\"",
    "start": "nest start",
    "start:dev": "nest start --watch",
    "start:debug": "nest start --debug --watch",
    "start:prod": "node dist/main",
    "lint": "eslint \"{src,apps,libs,test}/**/*.ts\" --fix",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:cov": "jest --coverage",
    "test:debug": "node --inspect-brk -r tsconfig-paths/register -r ts-node/register node_modules/.bin/jest --runInBand",
    "test:e2e": "jest --config ./test/jest-e2e.json"
  }
}
```

The `name` field, `rate-limit`, is the only place in the whole project where the project's purpose is named directly, and it matches the folder name of this repo, `Rate-Limit-in-NestJS-using-Throttler-main`. `private: true` and `license: UNLICENSED` simply mean this package is not meant to ever be published to npm, which is normal for an application (as opposed to a reusable library).

The scripts are the standard set the Nest CLI generates for every new project. `npm run start` boots the app once, `npm run start:dev` boots it in watch mode so it restarts automatically whenever you save a file, and `npm run start:prod` runs the already compiled JavaScript straight out of `dist` with plain Node, which is what you would actually run on a real server. `npm run test` runs every `*.spec.ts` file found under `src` using Jest, and `npm run test:e2e` runs Jest again but pointed at a separate configuration file, `test/jest-e2e.json`, which only picks up files ending in `.e2e-spec.ts` inside the `test` folder. Both of these test scripts are covered in full in [05-testing-and-recap.md](05-testing-and-recap.md).

## The dependency that matters most here

```json
"dependencies": {
  "@nestjs/common": "^11.0.1",
  "@nestjs/core": "^11.0.1",
  "@nestjs/platform-express": "^11.0.1",
  "@nestjs/throttler": "^6.4.0",
  "reflect-metadata": "^0.2.2",
  "rxjs": "^7.8.1"
}
```

`@nestjs/common`, `@nestjs/core`, and `@nestjs/platform-express` are the three packages every NestJS application needs no matter what it does, they provide the decorators, the dependency injection container, and the Express based HTTP server underneath. `reflect-metadata` and `rxjs` are two supporting libraries Nest itself depends on internally (`reflect-metadata` lets Nest read type information off your classes at runtime for dependency injection to work, `rxjs` provides the Observable type Nest's internals are built around).

`@nestjs/throttler`, pinned at `^6.4.0`, is the one dependency in this list that is not part of every Nest project, and it is the entire reason this repository exists. It is NestJS's official rate limiting package, and every concept covered in the rest of these notes, `ThrottlerModule`, `ThrottlerGuard`, `Throttle`, and `seconds`, comes from this single package. The caret in `^6.4.0` means npm is free to install any compatible 6.x version at or above 6.4.0, following normal semantic versioning rules.

The `devDependencies` block below it is entirely ordinary tooling for a TypeScript Nest project, `typescript` itself, `ts-jest` and `jest` for running tests, `@nestjs/cli` and `@nestjs/schematics` for the `nest` command line tool, `eslint` and `prettier` for linting and formatting, and `supertest` for making HTTP requests inside the end to end test. None of it is specific to rate limiting, so there is nothing more to say about it here.

Notice also the `jest` block embedded directly inside `package.json` (rather than in a separate `jest.config.js` file), which sets `rootDir` to `src` and `testRegex` to match any file ending in `.spec.ts`. This is what makes `npm run test` find `src/app.controller.spec.ts` automatically.

## `main.ts`, how the application actually starts

```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

This is the entry point of the whole application, and it is as plain as a Nest entry point gets. `NestFactory.create(AppModule)` reads `AppModule` (covered in the next two notes) and builds the entire dependency injection container from it, wiring up every controller, provider, and imported module described there. `await app.listen(process.env.PORT ?? 3000)` then starts the underlying HTTP server. `process.env.PORT ?? 3000` reads the `PORT` environment variable if one has been set (useful when deploying somewhere that assigns its own port), and falls back to `3000` with the `??` nullish coalescing operator if `PORT` is undefined. There is nothing rate limiting specific in this file at all, throttling is entirely configured inside `AppModule`, which is exactly why the next two notes focus there.

## TypeScript and Nest CLI configuration

`tsconfig.json` sets the compiler options shared by the whole project:

```json
{
  "compilerOptions": {
    "module": "commonjs",
    "declaration": true,
    "removeComments": true,
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "allowSyntheticDefaultImports": true,
    "target": "ES2023",
    "sourceMap": true,
    "outDir": "./dist",
    "baseUrl": "./",
    "incremental": true,
    "skipLibCheck": true,
    "strictNullChecks": true,
    "forceConsistentCasingInFileNames": true,
    "noImplicitAny": false,
    "strictBindCallApply": false,
    "noFallthroughCasesInSwitch": false
  }
}
```

Two of these options matter more than the rest for understanding how Nest works at all. `experimentalDecorators: true` turns on support for the `@Controller()`, `@Injectable()`, `@Module()`, and `@Throttle()` style decorators used throughout this project, and `emitDecoratorMetadata: true` makes the compiler also emit extra type information alongside those decorators, which is exactly what lets Nest's dependency injection system look at a constructor parameter like `private readonly appService: AppService` and know, purely from the type annotation, which class to hand it at runtime. Without both of these settings turned on, none of the decorator based code in this project would work.

The rest of the options are fairly standard, `target: "ES2023"` compiles down to a recent but still safely supported version of JavaScript, `outDir: "./dist"` is where compiled files end up (which is exactly what `npm run start:prod` runs), and `strictNullChecks: true` combined with `noImplicitAny: false` reflects a middle ground level of strictness common in Nest starter templates, catching null and undefined mistakes while not forcing every implicit value to be explicitly typed.

`tsconfig.build.json` extends that same file and simply excludes test files and `node_modules` from a production build:

```json
{
  "extends": "./tsconfig.json",
  "exclude": ["node_modules", "test", "dist", "**/*spec.ts"]
}
```

`nest-cli.json` tells the Nest CLI tool where your source code lives and how to build it:

```json
{
  "$schema": "https://json.schemastore.org/nest-cli",
  "collection": "@nestjs/schematics",
  "sourceRoot": "src",
  "compilerOptions": {
    "deleteOutDir": true
  }
}
```

`sourceRoot: "src"` tells commands like `nest generate` where to create new files, and `deleteOutDir: true` means every time you run `nest build`, the old `dist` folder is wiped clean first so you never end up running stale compiled output by accident.

## Linting and formatting

`.prettierrc` is a tiny two line configuration, `{ "singleQuote": true, "trailingComma": "all" }`, which is why every string in this codebase uses single quotes and every multi line array or object ends with a trailing comma. `eslint.config.mjs` wires together ESLint's own recommended rules, the type aware `typescript-eslint` rules, and Prettier integration, then loosens a few of them for practical reasons, `@typescript-eslint/no-explicit-any` is turned off entirely, and `@typescript-eslint/no-floating-promises` and `@typescript-eslint/no-unsafe-argument` are downgraded from errors to warnings. None of this changes how the application behaves at runtime, it only affects what the linter complains about while you are writing code, but it is worth knowing this project favors a slightly relaxed linting setup rather than the strictest possible configuration.

With the setup out of the way, the next note moves on to the actual subject of this project, what rate limiting is and how `ThrottlerModule` configures it. See [02-what-is-rate-limiting-and-throttlermodule.md](02-what-is-rate-limiting-and-throttlermodule.md).
