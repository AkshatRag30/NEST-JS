# 2. Project Setup and Structure

## The npm scripts that run this project

Open `package.json` and look at the `scripts` block:

```json
"scripts": {
  "build": "nest build",
  "start": "nest start",
  "start:dev": "nest start --watch",
  "start:debug": "nest start --debug --watch",
  "start:prod": "node dist/main",
  "lint": "eslint \"{src,apps,libs,test}/**/*.ts\" --fix",
  "test": "jest",
  "test:e2e": "jest --config ./test/jest-e2e.json"
}
```

As a beginner, the two you will use constantly are `npm run start:dev` and `npm run test`.

`npm run start:dev` runs `nest start --watch`. The `nest` command comes from the `@nestjs/cli` package (a dev dependency). `--watch` means the process keeps running and automatically restarts the server every time you save a file, so you do not have to stop and re-run it manually while developing.

`npm run build` compiles all your TypeScript files in `src` into plain JavaScript inside a `dist` folder. `npm run start:prod` then runs the compiled JavaScript directly with plain Node (`node dist/main`), which is what you would actually deploy, since production servers do not need to compile TypeScript on the fly.

`npm run test` runs Jest against every file that matches `*.spec.ts` (this is configured inside the `jest` block of `package.json`). `npm run test:e2e` runs a separate configuration (`test/jest-e2e.json`) meant for end to end tests that spin up the whole application. Both are covered fully in [17-testing-in-nestjs.md](17-testing-in-nestjs.md).

## Dependencies versus devDependencies

Look at the two blocks in `package.json`.

`dependencies` are packages needed while the app is actually running: `@nestjs/common`, `@nestjs/core`, `@nestjs/platform-express` (the core framework), `@nestjs/config` (for environment variables), `class-validator` and `class-transformer` (for validating incoming data), `reflect-metadata` (required internally by NestJS's decorator and dependency injection system), and `rxjs` (Reactive Extensions for JavaScript, used internally by NestJS for things like guards that can return an `Observable`).

`devDependencies` are packages only needed while developing: the CLI, TypeScript itself, ESLint and Prettier (code style tools), and Jest plus Supertest (testing tools).

## `nest-cli.json` and `tsconfig.json`

`nest-cli.json` tells the Nest CLI where your source code lives:

```json
{
  "sourceRoot": "src",
  "compilerOptions": { "deleteOutDir": true }
}
```

`sourceRoot: "src"` is why every feature lives inside the `src` folder. `deleteOutDir: true` means every time you build, the old `dist` folder is wiped first so stale compiled files never linger around.

`tsconfig.json` configures the TypeScript compiler itself. Two settings matter a lot for how NestJS works under the hood:

```json
"emitDecoratorMetadata": true,
"experimentalDecorators": true
```

These two flags are what make decorators like `@Injectable()` and `@Controller()` possible at all. `experimentalDecorators` turns on decorator syntax support in TypeScript. `emitDecoratorMetadata` makes TypeScript also emit type information about constructor parameters into the compiled JavaScript, which is exactly how NestJS's dependency injection knows what class to inject into a constructor without you writing any extra configuration. This connects directly to [07-dependency-injection.md](07-dependency-injection.md), so keep this in the back of your mind.

## The folder structure of `src`

Here is what actually exists in this repo's `src` folder, grouped by what it demonstrates:

Root level files: `main.ts` (entry point), `app.module.ts` (root module), `app.controller.ts` and `app.service.ts` (the default "Hello World" feature that ships with every new Nest project).

Feature folders that are full standalone modules (each with its own `*.module.ts`): `category`, `customer`, `employee`, `student`. These are the "properly modularized" examples.

Feature folders registered directly on the root `AppModule` instead of having their own module file: `product`, `user`, `myname`, `user-roles`, `exception`, `database`, `ev`. These exist to keep some examples simple while you are still learning, and to show you that a module is not strictly mandatory for a tiny controller/service pair (though in real projects you almost always give every feature its own module, see [04-modules.md](04-modules.md) for why).

Cross cutting concern folders, meaning code that is not tied to one feature but applies across many: `common/pipes/uppercase` (a custom pipe), `filters/http-exception` (a custom exception filter), `guards/auth` and `guards/roles` (two different guard examples), `middleware/logger` (a custom middleware).

## The entry point: `main.ts`

Every NestJS application starts execution here:

```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { ValidationPipe } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.useGlobalPipes(new ValidationPipe({
    whitelist: true,
    forbidNonWhitelisted: true
  }))
  await app.listen(process.env.PORT ?? 3000);
  app.enableShutdownHooks();
}
bootstrap();
```

Reading this line by line, in plain language:

`NestFactory.create(AppModule)` builds the entire application. It reads `AppModule`, discovers everything it imports (other modules, controllers, providers), builds the dependency injection container, and hands you back an `app` object representing the whole running application. This is the single most important line in any NestJS app, because it is the trigger that turns your `@Module`, `@Controller`, and `@Injectable` classes from plain TypeScript into a running, wired up web server.

`app.useGlobalPipes(new ValidationPipe({...}))` registers a pipe that runs on every single incoming request across the whole app (as opposed to a pipe applied to just one route). `whitelist: true` means any property on an incoming request body that is not declared in your DTO gets silently stripped out. `forbidNonWhitelisted: true` goes further: instead of silently stripping unknown properties, it rejects the request with an error if it contains any property you did not declare. This is explained fully in [10-validation-and-pipes.md](10-validation-and-pipes.md).

`app.listen(process.env.PORT ?? 3000)` starts the actual HTTP server. `process.env.PORT` reads an environment variable named `PORT`; if it is not set, `?? 3000` (the nullish coalescing operator) falls back to port 3000. This is your first taste of environment variables, expanded on in [15-environment-variables-and-config.md](15-environment-variables-and-config.md).

`app.enableShutdownHooks()` tells NestJS to actually call your lifecycle shutdown hooks (like `onApplicationShutdown`) when the process receives a termination signal (for example when you press Ctrl+C, or a hosting platform stops the container). Without this line, methods like the one in `src/database/database.service.ts` would never run. Full detail in [14-lifecycle-events.md](14-lifecycle-events.md).

`bootstrap()` at the very bottom is just calling the function you just defined, kicking the whole process off. `bootstrap` is only a name, by convention, not a special keyword.

## What to do next

Read [03-architecture-overview.md](03-architecture-overview.md) next. It zooms out from `main.ts` and explains, at a conceptual level, how a request actually travels through a NestJS app once it is running, and how the pieces you saw scattered across the `src` folder relate to each other.
