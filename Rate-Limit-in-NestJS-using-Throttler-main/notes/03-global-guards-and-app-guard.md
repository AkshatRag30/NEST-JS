# 03. Global Guards and the APP_GUARD Pattern

## A quick reminder of what a guard is

A guard in NestJS is a class that decides whether a given request is allowed to continue on to its route handler or should be stopped right there. Every guard implements the `CanActivate` interface, meaning it has a `canActivate` method that returns `true` to let the request through or `false` (or throws an exception) to block it. The most common way to attach a guard to a route is the `@UseGuards()` decorator placed directly above a controller or a single route handler. This project, however, does not use `@UseGuards()` anywhere at all, and understanding why is the whole point of this note.

## The guard that actually enforces throttling

`@nestjs/throttler` ships its own guard, `ThrottlerGuard`, which is the class that actually reads the configuration set up by `ThrottlerModule.forRoot()` (covered in [02-what-is-rate-limiting-and-throttlermodule.md](02-what-is-rate-limiting-and-throttlermodule.md)) and enforces it on real incoming requests. Configuring `ThrottlerModule.forRoot()` alone only describes the rules, `limit: 3`, `ttl: seconds(60)`, and so on, it is `ThrottlerGuard` that actually counts requests per client and rejects the ones that go over.

## How this project applies it to every route at once

```ts
providers: [AppService, {
  provide: APP_GUARD,
  useClass: ThrottlerGuard
}],
```

This is the `providers` array from `app.module.ts`. Alongside the ordinary `AppService` provider sits an object with two keys, `provide: APP_GUARD` and `useClass: ThrottlerGuard`. This is a special pattern worth explaining slowly, because it is not obvious the first time you see it and it looks nothing like `@UseGuards()`.

`APP_GUARD` is a constant exported by `@nestjs/core`, and it is not an ordinary class you inject and use like `AppService`. It is a specific, reserved injection token that NestJS itself recognizes as meaningful. Normally, when you list something in a `providers` array, you are just making a class available for other classes in that module to have injected into their constructors. But when the token you provide happens to be `APP_GUARD` specifically, NestJS treats the class you registered under it differently, it automatically registers that class as a guard that runs on every single route in the entire application, with no `@UseGuards()` needed anywhere. This is NestJS's standard mechanism for applying a guard globally, and `APP_GUARD` has two close relatives that work the same way for other request pipeline pieces, `APP_INTERCEPTOR` for interceptors and `APP_FILTER` for exception filters.

The `useClass: ThrottlerGuard` half of that object tells Nest which class to actually construct and use for this global guard, following the same `provide`/`useClass` shape you would use for any custom provider registration. Because it is registered through Nest's regular dependency injection system rather than being manually instantiated with `new ThrottlerGuard()` somewhere, Nest is also free to give `ThrottlerGuard` its own injected dependencies (in particular, the throttling configuration and storage that `ThrottlerModule.forRoot()` set up), the exact same dependency injection machinery covered for ordinary services elsewhere in NestJS applies here too.

## Why this matters for a beginner

Without this `APP_GUARD` registration, configuring `ThrottlerModule.forRoot()` alone would do nothing, the rules would exist but nothing would actually be checking incoming requests against them, and you would need to manually add `@UseGuards(ThrottlerGuard)` above every single controller or route you wanted protected, one at a time, and remember to keep doing it for every new route added later. By registering `ThrottlerGuard` once under `APP_GUARD` in `app.module.ts`, this project guarantees that every route, current and future, is covered automatically. In this particular repository there is only one route to protect, the single `GET` route in `app.controller.ts`, so the practical difference is small here, but this pattern is exactly how you would want to apply throttling (or authentication, or any other global check) across a real application with dozens of routes across many controllers, without having to remember to decorate each one individually.

The next note looks at something that might seem redundant at first glance, given that the guard is already global, this project's one route also carries its own `@Throttle()` decorator. See [04-per-route-throttle-overrides.md](04-per-route-throttle-overrides.md) for what that decorator actually does and whether it changes anything here.
