# 09. Testing Gaps, Verified by Actually Running Them

Every service and resolver in this repository has a `.spec.ts` file sitting right next to it, and just like the MongoDB sibling project, reading them closely reveals real problems. This project's problems turned out to run deeper than that sibling's did, and rather than only reading the code and predicting what would happen, every claim in this note was checked directly: dependencies were installed, the Prisma client was generated, and the actual test suite and a real production style build were both run for real, with `DATABASE_URL` deliberately left unset, the same condition anyone freshly cloning this repository would be in.

## The deeper bug: a broken import path, not just a missing test provider

Look closely at two imports in this codebase. In `book.module.ts`:

```ts
import { PrismaModule } from 'src/prisma/prisma.module';
```

And in `book.service.ts`:

```ts
import { PrismaService } from 'src/prisma/prisma.service';
```

And in `prisma.service.ts`:

```ts
import { PrismaClient } from 'generated/prisma';
```

All three of these are written as bare, non relative specifiers, `'src/prisma/prisma.module'` rather than `'./prisma.module'` or `'../prisma/prisma.module'`. As explained in [01-project-setup-and-bootstrap.md](01-project-setup-and-bootstrap.md), these only type check successfully at all because `tsconfig.json` sets `baseUrl: "./"`, which tells the TypeScript compiler to also try resolving a bare specifier against the project root. But `baseUrl` only affects TypeScript's own type checking step, it does not rewrite these paths into real relative paths in the JavaScript that actually gets emitted or run. Compare this against `app.module.ts`, which imports the exact same `PrismaModule` correctly: `import { PrismaModule } from './prisma/prisma.module';`, a genuine relative path that works everywhere, with no special configuration needed.

This was verified directly, not assumed. After running `npm install`, generating the Prisma client with `npx prisma generate`, and compiling the project with `npx tsc -p tsconfig.build.json` (exactly what `npm run build` does), the compiled output at `dist/book/book.service.js` contains this line, completely unchanged from the source:

```js
const prisma_service_1 = require("src/prisma/prisma.service");
```

Attempting to load that compiled file directly with plain Node.js reproduces a real, immediate crash:

```
Error: Cannot find module 'src/prisma/prisma.service'
Require stack:
- dist/book/book.service.js
```

The same is true of `dist/prisma/prisma.service.js`, which contains `require("generated/prisma")` unchanged, and would fail to resolve for the identical reason the moment anything tries to load it. Since `book.module.ts` imports `PrismaModule`, and `AppModule` imports `BookModule`, this means the fully built, production style version of this application (`npm run build` followed by `npm run start:prod`) cannot actually start at all, it crashes on a module resolution error before it ever gets anywhere near the missing `DATABASE_URL` problem covered in earlier notes. Neither `package.json`'s scripts nor any config file in this repository sets up the piece that would normally fix this, something like the `tsconfig-paths` package's runtime loader (`ts-node -r tsconfig-paths/register`, mentioned only inside the unused `test:debug` script) or a `paths` mapping paired with a matching bundler resolution plugin.

## What this means for the unit test suite, confirmed by actually running it

Running `npx jest` in this project, with `DATABASE_URL` unset, against all four `.spec.ts` files, produces exactly this real result:

```
FAIL src/prisma/prisma.service.spec.ts
  ● Test suite failed to run
    Cannot find module 'generated/prisma' from 'prisma/prisma.service.ts'

PASS src/app.controller.spec.ts

FAIL src/book/book.service.spec.ts
  ● Test suite failed to run
    Cannot find module 'src/prisma/prisma.service' from 'book/book.service.ts'

FAIL src/book/book.resolver.spec.ts
  ● Test suite failed to run
    Cannot find module 'src/prisma/prisma.service' from 'book/book.service.ts'

Test Suites: 3 failed, 1 passed, 4 total
Tests:       1 passed, 1 total
```

Three of the four spec files fail before a single test inside them ever runs, and all three fail for the same root cause, the broken import path above, not because of anything specific to what each spec file itself does or does not provide to `Test.createTestingModule(...)`.

`src/app.controller.spec.ts` is the one file that genuinely passes, and reading it explains why:

```ts
const app: TestingModule = await Test.createTestingModule({
  controllers: [AppController],
  providers: [AppService],
}).compile();
```

`AppController` and `AppService` are both untouched Nest CLI scaffold code with no Prisma or GraphQL involvement at all, `AppService.getHello()` simply returns the literal string `'Hello World!'`, and the spec asserts exactly that string. Unlike the MongoDB sibling project, where the equivalent scaffold test had actually drifted out of sync with a hand edited `AppService`, this project's `app.controller.spec.ts` still matches the real implementation exactly, and it is the only spec file in this repository that currently passes.

## What would still be wrong even if the import paths were fixed

It is worth separating the two categories of problem clearly, since fixing the import path bug above would not make every remaining spec file pass.

`book.service.spec.ts` and `book.resolver.spec.ts` fall into the exact same category of problem documented in the MongoDB sibling project's testing notes:

```ts
const module: TestingModule = await Test.createTestingModule({
  providers: [BookService],
}).compile();
```

```ts
const module: TestingModule = await Test.createTestingModule({
  providers: [BookResolver],
}).compile();
```

`BookService`'s constructor requires a `PrismaService`, and `BookResolver`'s constructor requires a `BookService`, and neither testing module supplies either dependency. Even with the module resolution bug fixed, `.compile()` would still fail on both of these, this time with Nest's own dependency injection error, along the lines of "Nest can't resolve dependencies of the BookService (?)", exactly the same category of scaffold code that was generated automatically and never updated once real constructor dependencies were added, covered in full (including the fix pattern using `getModelToken` style stand ins) in the MongoDB project's own testing notes.

`prisma.service.spec.ts` is genuinely different, and this was also checked directly rather than assumed, by reproducing the same test logic in isolation with the import path issue sidestepped. `PrismaService` itself declares no constructor of its own, it simply extends `PrismaClient` and adds lifecycle methods, so there is no injection token for `Test.createTestingModule({ providers: [PrismaService] }).compile()` to fail on. It was also confirmed directly that `TestingModule.compile()` does not automatically invoke `onModuleInit()` the way a full running Nest application would, so `$connect()` never actually gets called during this test, and the missing `DATABASE_URL` problem covered in [03-the-prisma-schema-file.md](03-the-prisma-schema-file.md) never has a chance to surface here at all. In other words, if the single broken import in `prisma.service.ts` were changed to a relative path, `prisma.service.spec.ts`'s one assertion, `expect(service).toBeDefined()`, would actually pass, cleanly, with no database connection ever attempted.

## The end to end test's own, separate exposure

`test/app.e2e-spec.ts` imports the real `AppModule` wholesale, exactly the same approach the MongoDB sibling project's e2e test uses:

```ts
const moduleFixture: TestingModule = await Test.createTestingModule({
  imports: [AppModule],
}).compile();

app = moduleFixture.createNestApplication();
await app.init();
```

Because this pulls in the real `BookModule`, which itself imports `PrismaModule` via the broken `'src/prisma/prisma.module'` specifier, this test would fail at exactly the same module resolution step as the unit tests above, before `app.init()` ever gets a chance to run, and therefore before the missing `DATABASE_URL` problem would ever become the actual, visible cause of failure here either. If that import were fixed, `app.init()` would then genuinely attempt to boot the real application, including `PrismaService.onModuleInit()`'s `$connect()` call, and would fail at that later point instead, with the exact `Environment variable not found: DATABASE_URL` error reproduced in [03-the-prisma-schema-file.md](03-the-prisma-schema-file.md), since this repository ships no `.env` file and no `DATABASE_URL` anywhere.

## Summary of what actually happens today

Running `npm run test` in this project as it currently stands produces three failing test suites and one passing one, and the failures are not primarily about missing test providers the way the MongoDB sibling project's were, they are a genuinely broken pair of import paths that would prevent this application from starting correctly even outside of any test, in a real deployment, with a perfectly valid `DATABASE_URL` already configured.
