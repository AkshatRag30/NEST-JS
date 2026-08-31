# 02. What Is Rate Limiting, and How ThrottlerModule Configures It

## What rate limiting actually is

Rate limiting means putting a cap on how many requests a single client is allowed to send to your API within a given window of time, and automatically rejecting any request that arrives once that cap has been used up, at least until the window resets. If a limit is set to three requests per minute, a client can send three requests right away, but a fourth request inside that same minute gets turned away with an error instead of being processed normally.

It is worth pausing on why an application would want this at all, before looking at any code. A public API has no control over who calls it or how carefully they have written their client code. Rate limiting protects against a few very real, very common situations. It protects against outright abuse, someone deliberately hammering your server with requests to try to overwhelm it or scrape data as fast as possible. It protects against brute force attempts, for example someone trying to guess a password or an API key by trying thousands of combinations back to back, since a tight limit makes that kind of guessing impractically slow. It also protects against something far more mundane and far more common in practice, an honest mistake in a client application, a bug that causes a frontend to retry a failed request in a tight loop, or a script someone wrote that accidentally fires the same request a thousand times in a second. Without any limit in place, any one of these situations can degrade or take down the server for every other user sharing it. Rate limiting is the mechanism that keeps one client's behavior, malicious or accidental, from being able to do that.

NestJS does not build this capability into its core framework, instead it ships an official, separate package for it, `@nestjs/throttler`, which this project depends on directly (see [01-project-setup-and-bootstrap.md](01-project-setup-and-bootstrap.md)). The rest of this note walks through exactly how this project configures it.

## `ThrottlerModule.forRoot()` in `app.module.ts`

```ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { seconds, ThrottlerGuard, ThrottlerModule } from '@nestjs/throttler';
import { APP_GUARD } from '@nestjs/core';

@Module({
  imports: [
    ThrottlerModule.forRoot({
      throttlers: [
        {
          name: 'default',
          ttl: seconds(60),
          limit: 3,
        }
      ],
      errorMessage: 'Too many requests! Please wait a minute and try again!',
    })
  ],
  controllers: [AppController],
  providers: [AppService, {
    provide: APP_GUARD,
    useClass: ThrottlerGuard
  }],
})
export class AppModule {}
```

This note focuses on the `imports` array, the `providers` array (the part that makes throttling apply to every route automatically) is its own topic and gets the full explanation it deserves in [03-global-guards-and-app-guard.md](03-global-guards-and-app-guard.md).

`ThrottlerModule` is imported from `@nestjs/throttler`, and calling its static `forRoot()` method is how you configure throttling for the whole application in one place, the same general pattern you would see with other Nest modules that need setup, like `ConfigModule.forRoot()` in other projects. `forRoot()` takes an options object, and the most important part of that object is `throttlers`, an array of named throttling profiles. This project defines exactly one, `name: 'default'`.

Each entry in `throttlers` needs a `limit` and a `ttl`. `limit: 3` means at most three requests are allowed per client within one window. `ttl` stands for time to live, and it defines how long that window lasts before the request count resets back to zero for that client. Here it is written as `seconds(60)` rather than a plain number.

## What the `seconds()` helper actually does

`seconds` is a small utility function exported by `@nestjs/throttler` whose entire job is to convert a plain number of seconds into the equivalent number of milliseconds, because internally the throttler package tracks time in milliseconds no matter how you write your configuration. `seconds(60)` evaluates to `60000`. Writing `seconds(60)` instead of just writing `60000` directly does not change the behavior at all, it exists purely to make configuration more readable for whoever is reading the code later, `seconds(60)` tells you at a glance that the window is sixty seconds long, while a bare `60000` forces you to remember, or go check, that the underlying unit is milliseconds. Put together, this `throttlers` entry means every client is limited to three requests per rolling sixty second window, tracked under the profile name `'default'`.

## `errorMessage`, the custom rejection text

```ts
errorMessage: 'Too many requests! Please wait a minute and try again!',
```

This sits alongside `throttlers` at the top level of the options object passed to `forRoot()`, rather than inside any one throttler entry, which means it applies as the default message across every throttler profile defined in this module (there is only the one profile here, but the option is not scoped to a specific profile). When a client goes over their limit, `@nestjs/throttler` normally responds with a generic message, and setting `errorMessage` here replaces that generic text with this project's own friendlier wording, which the client will see in the body of the error response once they have been rate limited.

## What happens when a client goes over the limit

Once `ThrottlerModule` is configured and the guard covered in the next note is actually wired in, any request beyond the configured `limit` within the current `ttl` window gets rejected automatically, before it ever reaches a controller method, with an HTTP `429 Too Many Requests` status code and the custom `errorMessage` text in the response body. The client that sent the first three requests within the minute gets normal responses, the fourth is the one that gets stopped. Configuring `ThrottlerModule.forRoot()` on its own only defines these rules, it does not yet apply them to any route by itself, that connection is made by the guard registration covered next, in [03-global-guards-and-app-guard.md](03-global-guards-and-app-guard.md).
