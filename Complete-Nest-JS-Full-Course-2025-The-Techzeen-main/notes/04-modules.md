# 4. Modules

## What a module actually is, at the code level

A module is a plain TypeScript class with an empty body, decorated with `@Module()`. All of the real information lives inside the object you pass to that decorator. Here is the simplest possible example from this repo, `src/employee/employee.module.ts`:

```ts
import { Module } from '@nestjs/common';
import { EmployeeService } from './employee.service';
import { EmployeeController } from './employee.controller';

@Module({
  providers: [EmployeeService],
  controllers: [EmployeeController]
})
export class EmployeeModule {
    
}
```

Notice the class body is empty. `EmployeeModule` does not have any methods or properties of its own (except in cases where a module implements lifecycle hooks, or the `NestModule` interface for middleware configuration, which you will see below). The class exists purely so NestJS has something to attach the `@Module()` metadata to, and so other modules have something to `import`.

## The four properties you can put inside `@Module()`

`controllers`: an array of controller classes that belong to this module. NestJS will instantiate these and start listening for the routes they declare.

`providers`: an array of classes (usually services) that belong to this module and should be available for dependency injection inside it.

`imports`: an array of other modules whose exported providers you want to use inside this module. This is how modules connect to each other.

`exports`: an array of providers from this module's own `providers` list that you want to make available to any other module that imports this one. If you do not export something, it stays private to this module even if another module imports it.

This repo's modules mostly only use `controllers` and `providers`, because most of the feature modules here are self-contained and do not need to share services with each other. `AppModule`, however, uses `imports` heavily, which is the more interesting case.

## The root module: `AppModule`

Every NestJS app needs exactly one entry point module, conventionally called `AppModule`, and it is the one you pass into `NestFactory.create()` in `main.ts`. Look at `src/app.module.ts`:

```ts
@Module({
  imports: [EmployeeModule, CategoryModule, StudentModule, CustomerModule, ConfigModule.forRoot({
    isGlobal: true,
  })],
  controllers: [AppController, UserController, ProductController, MynameController, UserRolesController, ExceptionController, DatabaseController, EvController],
  providers: [AppService, ProductService, DatabaseService, EvService],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(LoggerMiddleware).forRoutes('*');
  }
}
```

This single class ties the whole application together. Let's take it apart piece by piece.

`imports: [EmployeeModule, CategoryModule, StudentModule, CustomerModule, ConfigModule.forRoot({ isGlobal: true })]` pulls in four feature modules that this project built as fully independent modules, plus a special module (`ConfigModule`) from a third party package. `ConfigModule.forRoot({...})` is a common pattern you will see with real NestJS packages: instead of just writing `ConfigModule`, you call a static method (`forRoot`) that lets you pass configuration options while importing it. `isGlobal: true` here means "once this is imported into the root module, make its exported provider (`ConfigService`) available everywhere in the app without needing to import `ConfigModule` again in every other module." This is expanded on in [15-environment-variables-and-config.md](15-environment-variables-and-config.md).

`controllers: [AppController, UserController, ProductController, ...]` registers a bunch of controllers directly on the root module instead of giving each of them their own dedicated module. This is a shortcut that is fine for a small learning project, but in a real production app, every feature would typically get its own module (like `CategoryModule` does), even if that module only contains one controller and one service. Doing so keeps the root module small and keeps each feature's code physically grouped together, which matters a lot once a codebase grows to dozens of features.

`providers: [AppService, ProductService, DatabaseService, EvService]` registers the corresponding services for those directly attached controllers.

`implements NestModule` plus the `configure(consumer: MiddlewareConsumer)` method is a completely different mechanism from `controllers`/`providers`/`imports`. This is how you attach middleware to routes, and it is covered fully in [12-middleware.md](12-middleware.md). The short version: `consumer.apply(LoggerMiddleware).forRoutes('*')` says "run `LoggerMiddleware` before every single route in this application" (`'*'` means "all routes").

## Why bother with feature modules at all

Look at the difference between how `category` and `product` are set up. `category` has its own `category.module.ts` and is imported into `AppModule`. `product` has no module file at all; its controller and service are just listed directly inside `AppModule`'s `controllers` and `providers` arrays.

Both approaches work and NestJS does not force you to give every feature a module. But feature modules exist for a real reason: as an app grows to have authentication, orders, payments, users, products, and a dozen other features, if everything lived in one giant `AppModule`, that file would become an unreadable list of fifty controllers and fifty providers. By giving each feature its own module, you get physical, enforced separation: everything related to "customer" lives inside the `customer` folder and is summarized by one small `CustomerModule` file, and `AppModule` only needs to say `imports: [CustomerModule]` instead of listing every single class that customer feature needs.

This also sets up scoping. By default, providers declared in one module are private to that module unless explicitly exported. This means two different features can each have their own internal helper class with the same name, and they will never collide, because each module only sees its own providers unless something is deliberately exported and imported.

## A full trace: how `CustomerModule` gets used

`src/customer/customer.module.ts`:

```ts
@Module({
  providers: [CustomerService],
  controllers: [CustomerController]
})
export class CustomerModule {}
```

`src/app.module.ts` imports it:

```ts
imports: [EmployeeModule, CategoryModule, StudentModule, CustomerModule, ...]
```

Because `CustomerModule` is imported into `AppModule`, and `AppModule` is the module passed to `NestFactory.create(AppModule)` in `main.ts`, NestJS will discover `CustomerModule` during startup, see that it declares `CustomerController`, register the routes on that controller (everything under `/customer`, since `@Controller('customer')` is declared on that class), and make `CustomerService` available for injection into `CustomerController`'s constructor. If `CustomerModule` were never imported anywhere, none of its routes would exist, even though the files would still be sitting there on disk.

## Key terms recap

Module: a class decorated with `@Module()` that groups controllers and providers as one unit and can import other modules.

Root module (`AppModule`): the one module every NestJS app must have, passed into `NestFactory.create()`.

Feature module: a module dedicated to one feature/domain of the app, like `CategoryModule` or `CustomerModule`.

`imports`: brings in another module's exported functionality.

`exports`: makes a provider from this module available to modules that import it.

`forRoot()`: a common convention for a static method on a module class that lets you pass configuration while importing it (used by `ConfigModule.forRoot({...})`).
