# 02. Connecting NestJS to MongoDB

## The full `app.module.ts`

```ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from '@nestjs/config';
import { MongooseModule } from '@nestjs/mongoose';
import { StudentModule } from './student/student.module';
import { UserModule } from './user/user.module';
import { EmployeeModule } from './employee/employee.module';
import { ProductModule } from './product/product.module';
import { LibraryModule } from './library/library.module';
import { ProjectModule } from './project/project.module';

@Module({
  imports: [
    ConfigModule.forRoot(),
    MongooseModule.forRoot(process.env.MONGO_URI!),
    StudentModule,
    UserModule,
    EmployeeModule,
    ProductModule,
    LibraryModule,
    ProjectModule,
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

This is the root module, the one place in the whole app where every feature module gets pulled together. Everything in this note is about the two lines at the top of `imports`.

## `ConfigModule.forRoot()`

`ConfigModule` comes from `@nestjs/config`, and calling `.forRoot()` with no arguments tells it to look for a `.env` file in the project root and load whatever key value pairs it finds into `process.env`, so that later code (in this same file, on the very next line) can read them with `process.env.SOMETHING`.

One thing worth noticing here, since it is a genuine gap in this project rather than something you would want to copy: `ConfigModule.forRoot()` is called without `{ isGlobal: true }`. In a typical NestJS project you would pass that option so that `ConfigService` (a wrapper class around `process.env` with type safety and validation support) can be injected into any feature module without re-importing `ConfigModule` everywhere. This repo never actually injects `ConfigService` anywhere, every single environment variable read in this codebase is done directly through the raw Node.js global `process.env`, so the missing `isGlobal` flag has no visible effect here, but it does mean this project is not really using `@nestjs/config`'s dependency injection features at all, just its file loading behavior as a side effect of import order.

## `MongooseModule.forRoot(process.env.MONGO_URI!)`

This is the actual database connection line, and it only works correctly because it sits after `ConfigModule.forRoot()` in the `imports` array; Nest processes imports in the order they are listed, so by the time this line runs, `.env` has already been loaded into `process.env`, and `process.env.MONGO_URI` has a real value to read (assuming a `.env` file with that key exists locally, see [01-project-setup-and-bootstrap.md](01-project-setup-and-bootstrap.md) for why this repo does not ship one).

`MongooseModule.forRoot(...)` is the `@nestjs/mongoose` equivalent of calling `mongoose.connect(...)` directly, except it does it inside Nest's module system, meaning the connection is established once, application wide, exactly one time when the app starts, and any feature module that later calls `MongooseModule.forFeature(...)` (covered in [04-forfeature-and-dependency-injection.md](04-forfeature-and-dependency-injection.md)) automatically shares that same connection rather than opening a new one.

The `!` after `process.env.MONGO_URI` is a TypeScript non null assertion. Normally, `process.env.MONGO_URI` has the type `string | undefined`, because TypeScript has no way to know at compile time whether that environment variable will actually be set. The `!` is the developer telling the compiler "trust me, this will not be undefined," which silences a type error but adds zero actual runtime safety. If `MONGO_URI` is genuinely missing, `MongooseModule.forRoot(undefined)` gets called, and Mongoose will throw a connection error at startup rather than TypeScript catching the mistake ahead of time. This is a common shortcut in small projects and demos; a more defensive version would validate the variable exists before this line runs, for example with `@nestjs/config`'s schema validation feature, which this project does not use.

## The feature module list

The remaining six imports, `StudentModule`, `UserModule`, `EmployeeModule`, `ProductModule`, `LibraryModule`, and `ProjectModule`, are exactly the six feature folders under `src`. Each one is a self contained demonstration of a different Mongoose modeling pattern, and each is covered in its own note later in this folder ([05-crud-with-mongoose-models.md](05-crud-with-mongoose-models.md) through [08-many-to-many-referencing.md](08-many-to-many-referencing.md)). Importing a module here is what makes its controller's routes reachable and its provider available for the dependency injection container to construct.
