# 01. Project Setup, and What Is Actually Here

## An honesty note before anything else

This is the most important thing to understand about this repo, so it comes first rather than at the end. Every file inside `src` was checked directly, `app.controller.ts`, `app.controller.spec.ts`, `app.module.ts`, `app.service.ts`, and `main.ts`, along with `test/app.e2e-spec.ts` and every configuration file in the project root. None of them contain any JWT related code. There is no auth module, no login route, no guard, no strategy, no token signing or verification logic, anywhere in this codebase. `package.json` was also checked directly, and its dependency list contains none of the packages a real JWT setup would need: no `@nestjs/jwt`, no `@nestjs/passport`, no `passport`, no `passport-jwt`, no `bcrypt` or `bcryptjs`. What you are looking at is the plain, default output of running the Nest CLI's project generator, with one PDF slide deck dropped into a `slides` folder next to it. The rest of this note walks through exactly what that default scaffold contains, since even a "nothing has been built yet" project is worth understanding file by file if you are new to NestJS.

## `package.json`

The project is named `jwt-app`, version `0.0.1`, and marked `"private": true`. Its `dependencies` block lists exactly four things: `@nestjs/common`, `@nestjs/core`, and `@nestjs/platform-express`, which together are the minimum needed to run any NestJS HTTP application at all, plus `reflect-metadata` and `rxjs`, two libraries NestJS itself depends on internally, `reflect-metadata` for reading decorator metadata at runtime (this is what makes dependency injection possible) and `rxjs` for the reactive, observable based utilities NestJS uses in a few of its lower level APIs. That is the entire runtime dependency list. Compare that to a project that actually implements JWT auth, which would additionally need a token library and a password hashing library, neither of which appears here.

The `devDependencies` block is the standard tooling every fresh Nest CLI project ships with: `@nestjs/cli` and `@nestjs/schematics` for the `nest` command line tool itself, `@nestjs/testing` for building test modules, `jest` and `ts-jest` for running tests, `supertest` for making HTTP requests against a running Nest app in end to end tests, `eslint` and `prettier` related packages for linting and formatting, and `typescript` itself along with `ts-node` and `tsconfig-paths` for running TypeScript directly during development. Nothing in this list is JWT specific either, it is all generic Nest and TypeScript project scaffolding.

The `scripts` block gives you `build` (runs `nest build`), `start`, `start:dev` (watch mode, the one you would actually use while developing), `start:debug`, `start:prod` (runs the already compiled output in `dist/main.js` directly with plain `node`), `lint`, and the three test scripts, `test`, `test:watch`, and `test:cov` for unit tests, plus `test:e2e` for the single end to end test described below. There is also a `jest` configuration block embedded directly inside `package.json`, which tells Jest to treat `src` as its root directory and to run any file matching `*.spec.ts` as a unit test.

## `main.ts`, the entry point

```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

This is the plainest possible way to start a Nest application, and it is worth reading slowly if you have not seen it before, since every NestJS app in the world starts this same way. `NestFactory.create(AppModule)` is the line that builds the entire dependency injection container: Nest reads the `@Module()` decorator on `AppModule`, looks at everything it declares as a controller or a provider, and instantiates all of it in the right order. `app.listen(process.env.PORT ?? 3000)` then starts the actual HTTP server. `process.env.PORT` reads an environment variable named `PORT`, and since this repo ships no `.env` file and no `@nestjs/config` setup at all, that variable will almost always be undefined when you run this locally, which means the `??` operator falls through to the hardcoded default of port `3000`. `bootstrap()` is declared as an `async` function specifically so it can `await` those two asynchronous steps, and it is then simply called once at the bottom of the file.

## `app.module.ts`, `app.controller.ts`, and `app.service.ts`

These three files are the textbook "Hello World" that `nest new` generates for every single new project, completely unmodified here. `app.module.ts` declares an `AppModule` with an empty `imports` array, one controller (`AppController`), and one provider (`AppService`). `app.controller.ts` defines a single route, a `GET` request to the root path `/`, which calls `this.appService.getHello()` and returns whatever comes back. `app.service.ts` defines that one method, `getHello()`, which does nothing more than return the literal string `'Hello World!'`. There is no routing beyond this one endpoint, no request body parsing, no validation, and, again, nothing related to login, tokens, or protected routes.

## The test files

`src/app.controller.spec.ts` is a unit test that builds a `TestingModule` containing just `AppController` and `AppService`, then asserts that calling `getHello()` returns `'Hello World!'`. `test/app.e2e-spec.ts` does the same thing at a higher level: it boots the entire `AppModule` inside a real (in memory) Nest application, then uses `supertest` to make an actual `GET /` HTTP request and asserts the response is a `200` status with the body `'Hello World!'`. `test/jest-e2e.json` is the Jest configuration used specifically for that end to end run, pointing Jest at any file matching `*.e2e-spec.ts` rather than the `*.spec.ts` pattern used for unit tests. Both test files, like everything else in `src`, are exactly what the Nest CLI generates by default; neither one exercises anything JWT related, because there is nothing JWT related to exercise.

## The configuration files

`tsconfig.json` sets the usual NestJS compiler options, most importantly `experimentalDecorators: true` and `emitDecoratorMetadata: true`, which together are what make every decorator you will see in any NestJS project, `@Module()`, `@Controller()`, `@Injectable()`, `@Get()`, actually work and let Nest's dependency injection system inspect a constructor's parameter types at runtime. `strictNullChecks: true` is also on, though nothing in this tiny codebase currently exercises it in an interesting way. `tsconfig.build.json` extends that same config but excludes `test`, `dist`, and any `*spec.ts` file, since those should not be part of the compiled production build. `nest-cli.json` just tells the Nest CLI that source files live under `src` and that the `dist` output folder should be deleted and rebuilt fresh on every build. `.prettierrc` sets two formatting preferences, single quotes and trailing commas everywhere they are syntactically allowed, and `eslint.config.mjs` wires up ESLint with TypeScript aware rules plus Prettier integration, turning off the "no explicit any" rule and downgrading a couple of promise related and unsafe argument rules from errors to warnings. `.gitignore` excludes the usual suspects, `node_modules`, `dist`, `coverage`, log files, and any `.env` file, standard practice for keeping secrets and machine specific files out of version control, even though this particular repo has no `.env` file to begin with since it has no configuration that would need one yet.

## The `slides` folder

The one folder in this repo that is not part of the generated Nest scaffold is `slides`, which contains a single PDF, `JWT Tokens & Authentication Vs Authorization.pdf`. That eight slide deck is the entire reason this project exists, and it is the source for the next two notes in this folder. It never gets referenced from any code, imported anywhere, or turned into a runnable feature, it is purely reading material sitting alongside an otherwise empty application shell.
