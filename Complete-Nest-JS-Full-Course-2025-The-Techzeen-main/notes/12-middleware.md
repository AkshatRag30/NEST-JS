# 12. Middleware

## What middleware is

Middleware is a function (or, in NestJS, often a class) that runs before a request reaches its controller. The slide deck's definition is exactly right: "Middleware runs before the request reaches the controller." This concept did not originate with NestJS; it comes straight from Express, and NestJS's middleware system is built directly on top of it, which is why the shape of a NestJS middleware class should look familiar if you have ever used Express.

## Building the `LoggerMiddleware`

`src/middleware/logger/logger.middleware.ts`:

```ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log(`[${req.method}] - [${req.originalUrl}]`)
    next();
  }
}
```

`implements NestMiddleware` requires this class to have a `use` method with exactly this signature: `(req, res, next)`. These three parameters are the classic Express middleware signature. `req` is the incoming request object (you can read its method, URL, headers, body from here). `res` is the response object (you could end the response early from here if you wanted to, for example to block a request). `next` is a function that you must call when your middleware is done, to signal "pass control forward to whatever should run after me."

The actual logic here is small but genuinely useful: `console.log(`[${req.method}] - [${req.originalUrl}]`)` prints something like `[GET] - [/student/1]` to the server console for every single request that comes in, which is an extremely common real-world use for middleware, request logging. Then `next()` is called, letting the request continue on its way to whatever comes next (eventually, a guard, then a pipe, then the controller).

The one detail that trips people up the most: if you forget to call `next()`, the request simply hangs forever. The client's browser or HTTP client will sit there waiting for a response that never comes, because nothing downstream ever got the signal to continue.

## Registering middleware: `configure()` and `MiddlewareConsumer`

Unlike guards and pipes, which get attached with decorators (`@UseGuards`, `@UsePipes`) directly above a controller or route, middleware is registered in a different place entirely: inside a module class, by implementing the `NestModule` interface. Look at `src/app.module.ts` again:

```ts
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(LoggerMiddleware).forRoutes('*');
  }
}
```

`implements NestModule` requires `AppModule` to have a `configure(consumer: MiddlewareConsumer)` method. `MiddlewareConsumer` is an object NestJS gives you specifically for wiring up middleware. `consumer.apply(LoggerMiddleware)` says "I want to use `LoggerMiddleware`," and `.forRoutes('*')` says "apply it to every route in the entire application." `'*'` is a wildcard meaning "match everything." You could instead pass a specific string like `'student'` to only log requests to routes under `/student`, or even pass a specific controller class to scope it to just that controller's routes.

Because this is registered on `AppModule`, and `AppModule` is the root module that pulls in every other module, `LoggerMiddleware` genuinely runs on every single request this application ever receives, no matter which controller or module ultimately handles it.

## Middleware's use cases, and why each one fits here specifically

The slide deck lists five common use cases for middleware, and it is worth connecting each one to why middleware, specifically, is the right tool, rather than a guard or a pipe:

Logging incoming requests, exactly what `LoggerMiddleware` does here, is a good fit for middleware because you want it to run unconditionally, for every request, regardless of which controller or route eventually handles it, and you do not need any route-specific decision-making, you just want a side effect (writing a log line) to happen.

Authentication tokens (checking a JWT) often start in middleware because decoding a token and attaching the resulting user information onto the request object is a generic operation useful to almost every route, though the actual allow/deny decision based on that decoded information is typically then handled by a guard, which runs after middleware and has access to more specific, route-level context (like the `@Roles()` metadata seen in [11-guards-and-authorization.md](11-guards-and-authorization.md)).

Request transformation, blocking or redirecting requests, and setting headers are all naturally "before anything else happens" concerns, which is exactly what middleware is positioned for, since it runs earliest in the whole pipeline.

## Middleware versus Guard, precisely

The slide deck's comparison table is worth committing to memory because it is a very common point of confusion for beginners, and a common interview question:

Middleware runs before the controller and is meant for common, generic tasks unrelated to any specific route's authorization rules, like logging or decoding a token into a usable shape.

A guard runs specifically to decide whether a route can be accessed, based on authentication or authorization (role checks), and it runs after middleware, with access to much richer, route-specific context through `ExecutionContext` and `Reflector` (see [11-guards-and-authorization.md](11-guards-and-authorization.md)), which plain Express-style middleware does not have.

The practical rule of thumb: if the logic applies broadly and does not need to know which specific controller method is about to run, middleware is usually the simpler choice. If the logic is "should this specific request be allowed to reach this specific route," a guard is the correct tool, because it can inspect route-level metadata that middleware fundamentally cannot.

## Key terms recap

Middleware: a function or class that runs before a request reaches its controller, following the classic `(req, res, next)` signature.

`NestMiddleware`: the interface a middleware class implements, requiring a `use(req, res, next)` method.

`next()`: the function a middleware must call to pass control forward; forgetting to call it hangs the request.

`MiddlewareConsumer`: the object passed to a module's `configure()` method, used to wire up which middleware applies to which routes.

`forRoutes('*')`: applies the middleware to every route in the application; you can also target a specific path string or controller class instead.
