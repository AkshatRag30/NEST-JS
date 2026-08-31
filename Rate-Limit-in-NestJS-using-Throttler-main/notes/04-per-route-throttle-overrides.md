# 04. Per Route Throttle Overrides, and a Real Units Inconsistency

## The `@Throttle()` decorator in `app.controller.ts`

```ts
import { Controller, Get } from '@nestjs/common';
import { AppService } from './app.service';
import { Throttle } from '@nestjs/throttler';

@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get()
  @Throttle({ default: { limit: 3, ttl: 60000} })
  getHello(): string {
    return 'This is a rate-limit route!'
  }
}
```

`Throttle` is another export from `@nestjs/throttler`, and it lets you override the throttling configuration for one specific route, rather than accepting whatever `ThrottlerModule.forRoot()` set up globally for every route. It takes an object whose keys are the names of the throttler profiles you want to override, matched against the `name` you gave each entry in the `throttlers` array back in `app.module.ts`. Here the key is `'default'`, which matches exactly the `name: 'default'` profile defined in the module (see [02-what-is-rate-limiting-and-throttlermodule.md](02-what-is-rate-limiting-and-throttlermodule.md)), and the value, `{ limit: 3, ttl: 60000 }`, supplies the `limit` and `ttl` this specific route should use instead of the global defaults for that named profile.

## How an override actually interacts with the global config

Since `ThrottlerGuard` is already registered globally through `APP_GUARD` (see [03-global-guards-and-app-guard.md](03-global-guards-and-app-guard.md)), every route in this app is already being checked against the `'default'` throttler profile without needing anything extra. `@Throttle()` does not add a second, separate layer of throttling on top of that, instead it replaces the numbers used for the named profile specifically on the route it is attached to. If this route's `@Throttle()` had specified different numbers, say `{ default: { limit: 10, ttl: 30000 } }`, this one route would allow ten requests every thirty seconds while every other route in the application (if there were others) kept following the module level default of three requests every sixty seconds. That is the real purpose of this decorator, letting one particularly sensitive route (say, a login endpoint) be locked down tighter than the rest of the app, or letting one particularly cheap, high traffic route be more lenient than the rest, all without touching the shared global configuration.

## A real inconsistency worth noticing

Look closely at how time is written in these two places. In `app.module.ts`:

```ts
ttl: seconds(60),
```

And in `app.controller.ts`:

```ts
ttl: 60000
```

Both of these mean exactly the same thing, sixty seconds, because `seconds(60)` (explained in [02-what-is-rate-limiting-and-throttlermodule.md](02-what-is-rate-limiting-and-throttlermodule.md)) simply evaluates to `60000` milliseconds. But they are written completely differently within the same small codebase, one uses the self documenting helper function, and the other hardcodes the raw millisecond count directly. If you only had `app.controller.ts` open and had never seen `seconds()` used elsewhere, `ttl: 60000` gives you no hint on its own that the unit is milliseconds rather than, say, seconds or minutes, you would have to already know that `@nestjs/throttler` measures `ttl` in milliseconds to read that line correctly. This is a small but genuine inconsistency in this codebase, not a hidden bug, since the numeric value is correct either way, but it is exactly the kind of inconsistency that becomes a real source of confusion in a bigger project, if a later route used `ttl: 60` intending sixty seconds but actually meaning sixty milliseconds, that route would throttle far more aggressively than intended, and the mistake would be easy to make specifically because this project already shows both a helper based style and a raw number style existing side by side. The safer habit, and the one worth learning from this observation, is to use `seconds()` (or its sibling helpers `minutes()`, `hours()`, and so on, which `@nestjs/throttler` also exports) everywhere, so every `ttl` value documents its own unit at the call site.

## Does this override actually change anything here

Worth stating plainly, since it is easy to miss, in this specific project the override does not change the effective behavior of this route at all. The global `'default'` profile is `{ limit: 3, ttl: seconds(60) }`, and the route level override is `{ limit: 3, ttl: 60000 }`, which is the exact same limit and the exact same window, just written with a different unit style. So this particular route ends up throttled identically whether or not the `@Throttle()` decorator is present, three requests per sixty second window either way. Its presence in this controller reads as a demonstration of the syntax, showing a beginner what a per route override looks like and how it is written, rather than something that meaningfully changes this specific application's behavior. If you wanted to actually see a difference, changing the route's `limit` to something like `1` while leaving the module's global default at `3` would make the override's effect immediately visible in practice.

Next, [05-testing-and-recap.md](05-testing-and-recap.md) looks at the two test files in this project, and at a real bug that causes both of them to fail as currently written.
