# 01. Project Setup and Bootstrap

## What this project actually is

The `package.json` names this project `mongodbnestapp`, and its whole purpose is to demonstrate `@nestjs/mongoose`, the official NestJS wrapper around the `mongoose` library, which is the standard way Node.js applications talk to a MongoDB database. Looking at the `dependencies` block in `package.json`, on top of the usual `@nestjs/common`, `@nestjs/core`, and `@nestjs/platform-express` that every Nest app needs, this project adds three MongoDB specific packages: `@nestjs/mongoose` (the NestJS integration layer), `mongoose` (the underlying library that actually talks to MongoDB), and `@nestjs/config` (used here just to load a `.env` file, covered in the next note).

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

This is the plainest possible Nest bootstrap, and it is worth reading line by line since it is the starting point of everything else in this repo.

`NestFactory.create(AppModule)` is where the entire dependency injection container gets built. Nest reads `AppModule`'s `@Module()` decorator (covered in the next note), walks through every module it imports, and instantiates every controller and provider in the right order, resolving constructor arguments as it goes. This single line is why, later on, a class like `StudentService` can simply ask for a `Model<StudentDocument>` in its constructor and have it appear fully wired, without any manual wiring code anywhere in this repo.

`app.listen(process.env.PORT ?? 3000)` starts the actual HTTP server. `process.env.PORT` reads an environment variable named `PORT`; if it is undefined (which it will be, since no `.env` file ships with this repo, more on that below), the `??` operator falls back to `3000`. So unless you set a `PORT` variable yourself, this app always listens on port 3000.

`bootstrap()` is called once at the bottom of the file with no `await` at the top level, because top level code in a CommonJS entry file cannot itself be `async`; the function is defined as `async` so it can `await` the two asynchronous steps inside it, and then it is simply invoked.

## `tsconfig.json`

The compiler options worth knowing about here are `experimentalDecorators: true` and `emitDecoratorMetadata: true`. Every single feature in this repo, `@Schema()`, `@Prop()`, `@InjectModel()`, `@Injectable()`, `@Controller()`, is a TypeScript decorator, and `emitDecoratorMetadata` is specifically what lets Nest's dependency injection system look at a constructor parameter's declared TypeScript type at runtime and know which provider to inject. Without that flag, none of the `@InjectModel()` wiring covered later in these notes would work.

`strictNullChecks: true` is also turned on, which is why you will see return types written explicitly as `Promise<Student | null>` in the service files rather than just `Promise<Student>`, TypeScript is actually enforcing that the code acknowledges a document lookup can come back empty.

## `nest-cli.json`

This just tells the Nest CLI that `sourceRoot` is `src` and to delete the `dist` output folder before every build (`deleteOutDir: true`). Nothing unusual here, this is the default scaffold the Nest CLI generates for every new project.

## npm scripts, from `package.json`

`npm run start` runs `nest build` then runs the compiled app once. `npm run start:dev` runs it in watch mode, restarting automatically on file changes, this is the one you would use while working through this repo. `npm run test` runs every `*.spec.ts` file under `src` with Jest (see [09-testing-gaps.md](09-testing-gaps.md) for an important caveat about several of these). `npm run test:e2e` runs the single end to end test in `test/app.e2e-spec.ts`, which spins up the whole `AppModule`, meaning it needs a real, reachable MongoDB connection to even start, since `AppModule` calls `MongooseModule.forRoot()` unconditionally.

## The `.gitignore`, and why there is no `.env` file in this repo

Open `.gitignore` and you will see `.env` listed under a "dotenv environment variable files" section. That is completely standard practice, since a `.env` file usually contains secrets or machine specific connection strings that should never be committed to source control. The direct consequence for you as a reader of this repo is that the `MONGO_URI` value referenced in `app.module.ts` (see the next note) is not defined anywhere in this codebase, you or whoever runs this project is expected to create their own local `.env` file with a real connection string before this app can start successfully.
