# 01. Project Setup and Bootstrap

## What this project actually is

Open `package.json` and the first thing worth noticing is the `name` field, `"nestjs-prisma-neon"`, which is a much more honest label than the folder name `PrismaORM-NeonDB-NestJS-main` on disk, and it tells you plainly what this whole repository is a demonstration of. The `dependencies` block lists the usual `@nestjs/common`, `@nestjs/core`, and `@nestjs/platform-express` that every Nest app needs, and then five more packages that tell you exactly what two things this project bolts together: `@prisma/client` and `prisma` (the database toolkit, `prisma` itself lives under `devDependencies` since it is a command line tool used at development and build time, while `@prisma/client` is the runtime library your actual application code imports), and `@nestjs/graphql`, `@nestjs/apollo`, `apollo-server-express`, and `graphql` (the GraphQL layer, the same kind of stack the sibling `GraphQL-with-NestJS-main` project uses for its own in memory `Book` API).

## The entry point, `main.ts`

```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

This is the plainest possible Nest bootstrap, identical in shape to the one in the MongoDB sibling project. `NestFactory.create(AppModule)` is where Nest reads `AppModule`'s `@Module()` decorator, walks every module it imports, and builds the entire dependency injection container, resolving every constructor argument in the process, which is exactly the machinery that later lets `BookService` simply ask for a `PrismaService` in its constructor and receive a fully working, already connected instance with no manual wiring anywhere in this codebase.

`app.listen(process.env.PORT ?? 3000)` starts the HTTP server. If the `PORT` environment variable is not set, which it will not be by default since this repo ships no `.env` file, the `??` operator falls back to `3000`. Worth noticing here, since the lifecycle notes in the Techzeen sibling project cover this in depth: this `main.ts` never calls `app.enableShutdownHooks()`. That call is what tells Nest to listen for OS signals like `SIGINT` and actually invoke `onModuleDestroy` hooks across the app when the process is asked to stop. Since it is missing here, `PrismaService`'s own `onModuleDestroy` (covered in the next note), the piece of code responsible for cleanly closing the database connection, will never actually run when you stop this app with Ctrl+C or a container orchestrator's shutdown signal. The connection will simply be dropped whenever the Node process itself dies, rather than closed gracefully first.

## `tsconfig.json`

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

`emitDecoratorMetadata` and `experimentalDecorators` are what let every decorator used in this project, `@Injectable()`, `@Module()`, `@ObjectType()`, `@Field()`, `@Resolver()`, actually work at runtime, the same as in every other project in this course. `strictNullChecks: true` is why you will see Prisma's generated client methods typed to acknowledge that a lookup can come back with nothing, exactly the same discipline the MongoDB project's `Promise<Student | null>` return types enforced.

The one setting worth slowing down on here is `baseUrl: "./"`. Setting a `baseUrl` tells TypeScript that, in addition to relative imports like `./book.service`, it should also try resolving any import that does not start with `./` or `../` against this base path. That is exactly why a line like `import { PrismaService } from 'src/prisma/prisma.service';`, which shows up in this codebase, type checks successfully at all: TypeScript resolves `src/prisma/prisma.service` against the project root, finds `src/prisma/prisma.service.ts`, and is satisfied. Whether that same import actually works once the code is compiled and run for real is a completely separate question, one this project gets wrong in a way that is fully documented, with reproduced error output, in [09-testing-gaps.md](09-testing-gaps.md).

## `tsconfig.build.json`

```json
{
  "extends": "./tsconfig.json",
  "exclude": ["node_modules", "test", "dist", "**/*spec.ts"]
}
```

Nothing unusual here, this is the standard Nest CLI scaffold, it extends the main `tsconfig.json` and simply excludes test files and build output folders from a production build.

## `nest-cli.json`, `.prettierrc`, and `eslint.config.mjs`

`nest-cli.json` just sets `sourceRoot` to `src` and `deleteOutDir: true`, the default scaffold every new Nest project gets. `.prettierrc` sets `singleQuote: true` and `trailingComma: "all"`, pure formatting preferences with no behavioral effect. `eslint.config.mjs` uses the modern flat config format, pulling in `typescript-eslint`'s recommended type checked rules and Prettier's recommended integration, and then explicitly turns three rules down: `@typescript-eslint/no-explicit-any` is turned off entirely, and `@typescript-eslint/no-floating-promises` and `@typescript-eslint/no-unsafe-argument` are downgraded from an error to a warning. That second pair matters a little more than it looks: `no-floating-promises` being only a warning means ESLint will not hard fail this project for forgetting to `await` or otherwise handle a Promise, which is worth knowing since several places in this codebase (covered in later notes) return Promises from Prisma calls without any additional error handling wrapped around them.

## The `.gitignore`, and why there is no `.env` file in this repo

```
# dotenv environment variable files
.env
.env.development.local
.env.test.local
.env.production.local
.env.local

...

/generated/prisma
```

Just like the MongoDB sibling project, `.env` is listed under a dotenv section, meaning a local `.env` file with real secrets is expected to exist on any machine that actually runs this app, but is never committed to source control. The direct, real consequence, checked by actually searching this repository for any file or string containing `DATABASE_URL`, is that the connection string `prisma/schema.prisma` needs (covered fully in the next note) is not defined anywhere in this codebase. Nobody who clones this repository can run it against a real database until they create their own `.env` file with a working Neon connection string in it.

The last line, `/generated/prisma`, is specific to this project and worth flagging here since it explains a detail that shows up repeatedly in later notes: this project configures Prisma to generate its client code into a custom folder at the project root, `generated/prisma`, rather than Prisma's more common default location inside `node_modules`. Because that folder is generated code, not something a person hand writes, it is correctly excluded from version control here, exactly the same reasoning as excluding `/dist` or `/node_modules` a few lines above it.

## npm scripts, from `package.json`

The `jest` block embedded in `package.json` configures unit tests: `rootDir: "src"`, matching any file ending in `.spec.ts`, run through `ts-jest`. `npm run test` runs that suite. `npm run test:e2e` instead runs `jest --config ./test/jest-e2e.json`, a separate configuration pointed at `test/app.e2e-spec.ts`, which spins up the real `AppModule` end to end. `npm run build` runs `nest build` (which, since `nest-cli.json` sets no `webpack` option, compiles this project with plain `tsc` under the hood), and `npm run start:prod` simply runs the compiled output with `node dist/main`. Whether that compiled output can actually be required successfully by plain Node.js turns out to be exactly the same open question raised by the `baseUrl` detail above, and it is answered directly, with real reproduced command output, in [09-testing-gaps.md](09-testing-gaps.md).

## The default, untouched `README.md`

Unlike the MongoDB and GraphQL sibling projects, this project's own `README.md` was never edited away from the default file the Nest CLI generates for every brand new project, it is still the generic "Nest framework TypeScript starter repository" boilerplate with links to Nest's own documentation, Discord, and deployment options. It contains no project specific instructions at all, not even a mention of Prisma, Neon, or needing a `DATABASE_URL`, which reinforces the same point made above: getting this project running for real is left entirely to whoever clones it, with no in repo guidance beyond the code itself.
