# 01. Project Setup and the Postgres Connection

## What this project actually is

`package.json` names this project `postgresqldb`. Looking at its `dependencies` block, on top of the usual `@nestjs/common`, `@nestjs/core`, and `@nestjs/platform-express` that every Nest app needs, this project adds exactly four packages that matter for the database and auth side of things: `@nestjs/typeorm` (the official NestJS integration layer for TypeORM), `typeorm` (the actual object relational mapper that talks to the database), `pg` (the low level PostgreSQL driver for Node.js that TypeORM uses under the hood to actually open a socket and speak the Postgres wire protocol), and `jsonwebtoken` (a plain Node library for creating and verifying JSON Web Tokens, used only inside the auth guard, covered in [07-supabase-auth-guard.md](07-supabase-auth-guard.md)). There is no Supabase client library anywhere in `package.json`, no `@supabase/supabase-js`, which matters later when you read how the auth guard actually works, it never calls out to Supabase over the network at all.

`@nestjs/config` is also listed, used the same way you would expect from any NestJS project that reads environment variables through `ConfigService` rather than raw `process.env`.

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

This is the plainest possible Nest bootstrap. `NestFactory.create(AppModule)` is where the entire dependency injection container gets built, Nest reads `AppModule`'s `@Module()` decorator, walks through everything it imports, and constructs every controller and provider in the right order. `app.listen(process.env.PORT ?? 3000)` starts the HTTP server on whatever `PORT` environment variable is set, falling back to `3000` if none is set. Nothing here is specific to Postgres, the actual database wiring lives entirely in `app.module.ts`.

## The full `app.module.ts`

```ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { UserModule } from './user/user.module';
import { ConfigModule } from '@nestjs/config';
import { TypeOrmModule } from '@nestjs/typeorm';
import { EmployeesModule } from './employees/employees.module';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true
    }),
    TypeOrmModule.forRoot({
      type: 'postgres',
      url: process.env.DATABASE_URL,
      autoLoadEntities: true,
      synchronize: true
    }),
    UserModule,
    EmployeesModule],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

## `ConfigModule.forRoot({ isGlobal: true })`

This registers `@nestjs/config`, and unlike the MongoDB sibling project (which calls `ConfigModule.forRoot()` with no options at all), this project passes `{ isGlobal: true }`. That option means `ConfigService` becomes injectable anywhere in the app, in any module, without that module needing to import `ConfigModule` itself first. You can see this pay off directly in `SupabaseAuthGuard`, which injects `ConfigService` in its constructor even though the `auth` folder never imports `ConfigModule` on its own, covered in [07-supabase-auth-guard.md](07-supabase-auth-guard.md). By default, `ConfigModule.forRoot()` looks for a `.env` file in the project root and loads whatever it finds into a form `ConfigService` can read.

## `TypeOrmModule.forRoot({...})`, the actual database connection

This is the line that connects the whole app to PostgreSQL, and it is worth reading option by option.

`type: 'postgres'` tells TypeORM which database driver to use, this is what makes TypeORM reach for the `pg` package under the hood rather than, say, a MySQL or SQLite driver.

`url: process.env.DATABASE_URL` is the connection string itself, a single string in the form `postgres://user:password@host:port/database`, read straight out of an environment variable named `DATABASE_URL`. This is the connection string style of configuration, the alternative TypeORM also supports is passing individual fields (`host`, `port`, `username`, `password`, `database`) separately, but this project uses the single combined URL form instead. Notice there is no `!` non null assertion here the way the MongoDB sibling project writes `process.env.MONGO_URI!`, so TypeScript will happily accept `url: undefined` at compile time without complaint, since the type of `url` on TypeORM's connection options already allows `string | undefined`.

`autoLoadEntities: true` is a convenience option specific to `@nestjs/typeorm`. Normally you would need to list every entity class in an `entities: [...]` array right here in `forRoot`. With `autoLoadEntities: true`, TypeORM instead automatically picks up every entity that gets registered anywhere in the app through `TypeOrmModule.forFeature([...])`, which is exactly how both `Employee` (in `employees.module.ts`) and `User` (in `user.module.ts`) end up known to the database connection without either of them being listed here by name.

`synchronize: true` tells TypeORM to automatically create and alter database tables to match your entity classes every time the app starts, no manual migration files needed. This is genuinely convenient while learning, it is why running this project against a real Postgres database would create an `employee` table and a `user` table for you with zero manual SQL, but it is also a setting you would never want left on in a real production app, because TypeORM will alter your live tables automatically to match whatever the entity classes currently say, which can silently drop or change columns as the code evolves. Nothing in this repo turns it off for a production build, there is no `NODE_ENV` check anywhere gating this option.

## The missing `DATABASE_URL`, confirmed by checking the repo directly

Searching this entire project for any `.env` file turns up nothing, there is no `.env`, `.env.example`, or any variant of it committed anywhere in the repository. `.gitignore` confirms this is deliberate:

```
# dotenv environment variable files
.env
.env.development.local
.env.test.local
.env.production.local
.env.local
```

This is exactly the same situation as the `MONGO_URI` gap in the MongoDB sibling project, just with a different variable name. `process.env.DATABASE_URL` will be `undefined` in this repo exactly as it ships, which means `TypeOrmModule.forRoot({ url: undefined, ... })` gets called the moment the app starts. TypeORM will attempt to open a connection with no real connection string to work from, and the app will fail to start with a connection error. To actually run this project, you or whoever clones it needs to create a local `.env` file yourself with a real `DATABASE_URL=postgres://...` line pointing at an actual reachable Postgres database, this repo gives you the wiring but not the database itself, by design.

There is a second environment variable with the same problem, `SUPABASE_JWT_SECRET`, read inside `SupabaseAuthGuard`, covered fully in [07-supabase-auth-guard.md](07-supabase-auth-guard.md).

## `tsconfig.json` and `nest-cli.json`

`tsconfig.json` turns on `emitDecoratorMetadata: true` and `experimentalDecorators: true`, the two compiler options every single decorator in this project depends on, `@Entity()`, `@Column()`, `@Injectable()`, `@Controller()`, and `@InjectRepository()` all rely on this metadata being emitted so NestJS's dependency injection container can look at a constructor parameter's declared type at runtime and know what to hand it. `strictNullChecks: true` is on, but `noImplicitAny` is explicitly set to `false`, a looser setting than a fully strict TypeScript project would use. `nest-cli.json` is the default scaffold, `sourceRoot` is `src`, and `deleteOutDir: true` clears the `dist` folder before every build, nothing unusual here.

## npm scripts

`npm run start:dev` runs the app in watch mode, restarting on file changes, this is the one you would reach for while working through this repo, though it will not stay running successfully without a real `DATABASE_URL` as explained above. `npm run test` runs every `*.spec.ts` file under `src` with Jest, see [08-testing-gaps.md](08-testing-gaps.md) for exactly which of these actually pass. `npm run test:e2e` runs the single end to end test in `test/app.e2e-spec.ts`, which imports the real `AppModule` and therefore also needs a real, reachable Postgres database to get past its setup step.
