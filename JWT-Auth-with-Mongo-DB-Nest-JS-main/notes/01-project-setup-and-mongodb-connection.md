# 01. Project Setup and the MongoDB Connection

## What is actually installed

The real dependencies in `package.json` tell you exactly what this project is built from, and it is worth reading them before touching any code:

```json
"dependencies": {
  "@nestjs/common": "^11.0.1",
  "@nestjs/config": "^4.0.2",
  "@nestjs/core": "^11.0.1",
  "@nestjs/jwt": "^11.0.0",
  "@nestjs/mongoose": "^11.0.3",
  "@nestjs/passport": "^11.0.5",
  "@nestjs/platform-express": "^11.0.1",
  "bcrypt": "^6.0.0",
  "mongoose": "^8.16.3",
  "passport": "^0.7.0",
  "passport-jwt": "^4.0.1",
  "reflect-metadata": "^0.2.2",
  "rxjs": "^7.8.1"
}
```

Every piece of the authentication story is present here as a real, installed package, not a stub. `@nestjs/mongoose` and `mongoose` give the app its database layer. `@nestjs/jwt` wraps the `jsonwebtoken` library behind a small NestJS friendly `JwtService`. `@nestjs/passport` and `passport` give NestJS its guard based integration with the Passport authentication middleware ecosystem, and `passport-jwt` is the specific Passport strategy that knows how to pull a bearer token off a request and verify its signature. `bcrypt` is the real, native `bcrypt` package (not the pure JavaScript `bcryptjs` alternative some tutorials use), used for hashing and comparing passwords. `@nestjs/config` supplies `ConfigService`, which is how this project reads environment variables like the Mongo connection string and the JWT signing secret. Worth noting on the dev side too, `@types/bcrypt` is present, confirming `bcrypt`'s real native module is what is actually wired in, not a placeholder.

This is a small, single feature project. There is only one feature module, `auth`, alongside the default `AppModule`, `AppController`, and `AppService` that the Nest CLI scaffolds into every new project and that were never modified here.

## `main.ts`, the bootstrap

```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

This is the plain, unmodified Nest CLI starter bootstrap. `NestFactory.create(AppModule)` builds the whole dependency injection container by reading `AppModule`'s `imports`, `controllers`, and `providers`, and everything this project does, connecting to MongoDB, registering the JWT module, wiring up the Passport strategy, all happens because of what `AppModule` imports, not because of anything extra in `main.ts`. `app.listen(process.env.PORT ?? 3000)` starts the HTTP server on whatever `PORT` environment variable is set, falling back to `3000` if it is not. It is worth noting up front that `main.ts` never calls `app.useGlobalPipes(new ValidationPipe())`, there is no global validation pipe anywhere in this project, a detail that matters later when we look at what happens if a client sends a malformed signup or login request (see [03-registration-and-login-flow.md](03-registration-and-login-flow.md)).

## `app.module.ts`, where the database connection and the auth feature both get wired in

```ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from '@nestjs/config';
import { MongooseModule } from '@nestjs/mongoose';
import { AuthModule } from './auth/auth.module';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true
    }),
   MongooseModule.forRoot(process.env.MONGO_URI!),
   AuthModule 
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

Three things happen in this single `imports` array, in order.

`ConfigModule.forRoot({ isGlobal: true })` turns on `@nestjs/config` for the whole application. Passing `isGlobal: true` means every other module in this app, most importantly `AuthModule`, can inject `ConfigService` into its own providers without having to import `ConfigModule` itself again. `ConfigService`'s job is to read values out of environment variables (and, if a `.env` file existed here, out of that file too, though this repository does not ship one, its `.gitignore` explicitly excludes `.env` and every `.env.*.local` variant). This project reads two environment values through this mechanism across its lifetime, `MONGO_URI` and `MY_JWT_SECRET`, and neither has a committed example file anywhere in this repo, so both must be supplied by whoever runs this project, there is nothing to copy from.

`MongooseModule.forRoot(process.env.MONGO_URI!)` is the call that actually opens the connection to MongoDB. `MongooseModule.forRoot()` is the root level, one time setup for the whole application's database connection, as opposed to `MongooseModule.forFeature()` (used inside `AuthModule`, covered in the next note), which registers individual schemas as models once that shared connection already exists. The argument here is read straight from `process.env.MONGO_URI`, not through `ConfigService`, and note the trailing `!`, that is TypeScript's non null assertion operator, it tells the compiler "trust me, this value will not be `undefined` at runtime," but it does nothing at runtime itself. If `MONGO_URI` is genuinely not set in the environment when the app starts, `process.env.MONGO_URI` evaluates to `undefined` regardless of the `!`, and `MongooseModule.forRoot(undefined)` will fail to establish a connection, so the app will not start cleanly without a real `MONGO_URI` set. This is a real, honest gap worth knowing up front rather than being confused by an unexplained crash the first time you try to run this project without a database configured.

`AuthModule` is the third import, and it is what actually brings authentication into the app. Everything covered in the rest of these notes, the `User` schema, the signup and login logic, the JWT signing, the Passport strategy, and the three HTTP routes, lives inside this one module, described in full starting with the next note.

## Why there is no explicit `MongooseModule.forFeature()` call here

You will not find a `forFeature()` call in `app.module.ts` itself, and that is expected, not a gap. `forFeature()` belongs at the feature module level, right next to the schema it registers, so it lives in `auth.module.ts` instead. `AppModule` only owns the one shared connection; each feature module is responsible for telling that shared connection which collections it needs to work with. This project only has one feature module, so there is only one `forFeature()` call in the whole codebase, and it is covered in the next note alongside the schema it registers.
