# 18. Request Lifecycle Recap

You now know every individual piece. This note puts them back together in the exact order they run, using a single, concrete example request, and then gives you a table mapping every folder in `src` back to the note that explains it, so this file can double as a map of the whole project.

## The exact order things run, in a real NestJS app

For any incoming HTTP request, NestJS runs things in this order:

1. Middleware, registered globally or for matching routes. In this project, `LoggerMiddleware` (see [12-middleware.md](12-middleware.md)) runs here, for every single route, because `AppModule` applies it with `forRoutes('*')`.
2. Guards, attached to the matched route or controller. In this project, `AuthGuard` and `RolesGuard` (see [11-guards-and-authorization.md](11-guards-and-authorization.md)) run here, but only on the specific routes that used `@UseGuards(...)`.
3. Interceptors (before the handler runs). This repo does not use any custom interceptors, so this step is mentioned here only for completeness, interceptors are a related but separate mechanism from guards and pipes, letting you run code both before and after a route handler, useful for things like transforming a response's shape or measuring how long a request took.
4. Pipes, applied to individual parameters (`@Param`, `@Body`, `@Query`) or globally. In this project, the global `ValidationPipe` (see [10-validation-and-pipes.md](10-validation-and-pipes.md)) runs here for every route, alongside any route-specific pipes like `ParseIntPipe` or the custom `UppercasePipe`.
5. The route handler itself, meaning your actual controller method (see [05-controllers.md](05-controllers.md)), which almost always just calls straight into a service method (see [06-services-and-providers.md](06-services-and-providers.md)).
6. Interceptors again (after the handler runs, transforming the outgoing response), again not used in this repo.
7. The response is sent back to the client, unless something threw an error anywhere in steps 1 through 5, in which case control jumps straight to step 8.
8. Exception filters, which only run if an error was thrown somewhere along the way. In this project, `HttpExceptionFilter` (see [13-exception-filters.md](13-exception-filters.md)) runs here, but only for routes on `ExceptionController`, where it was applied with `@UseFilters(...)`. Everywhere else in this app, NestJS's own default exception handling formats the error response instead.

## Walking through one real request from this repo, start to finish

Take `GET /exception/hello/abc`, hitting `src/exception/exception.controller.ts`.

The request arrives at the server. `LoggerMiddleware` runs first (because it applies to `'*'`), printing `[GET] - [/exception/hello/abc]` to the console, then calling `next()` to let the request continue.

NestJS matches the URL to `ExceptionController`'s `getHello` method. This route has no `@UseGuards(...)` attached, so no guard logic runs, the request moves straight through this stage.

The global `ValidationPipe` has nothing to check here (there is no DTO-typed body on this route), so it passes through with no effect. Then the parameter-specific pipe kicks in: `@Param('id', ParseIntPipe)` tries to convert the string `"abc"` into a number. It cannot, so `ParseIntPipe` throws a `BadRequestException` right here, and `getHello`'s method body never actually executes.

Because an exception was thrown, execution jumps to exception filter handling. `ExceptionController` has `@UseFilters(HttpExceptionFilter)` on it, so this specific custom filter catches the error, builds its own response shape (`statusCode`, `timestamp`, `path`, `message`), and sends that back to the client as the final HTTP response.

Now take a second example, `POST /customer` with a valid body `{"name": "Zara", "age": 22}`, hitting `src/customer/customer.controller.ts`.

`LoggerMiddleware` runs and logs the request. There is no guard on this route, so that step is skipped. The global `ValidationPipe` inspects the body against `CreateCustomerDto`, confirms `name` is a string and `age` is an integer, and lets it through, converting the plain object into a real `CreateCustomerDto` instance in the process (see [08-dto-and-interfaces.md](08-dto-and-interfaces.md) and [10-validation-and-pipes.md](10-validation-and-pipes.md)). `addCustomer(createCustomerDto)` runs, calling straight into `this.customerService.addCustomer(createCustomerDto)`, which generates an `id`, pushes the new customer onto its in-memory array, and returns the newly created customer object. That object flows back up through the controller and becomes the JSON response sent to the client, with a `201 Created` status by default (NestJS's default status code for a successful `POST`).

## A map of every folder in `src` to the concept it teaches

| Folder / file | Concept | Note |
|---|---|---|
| `main.ts` | Application bootstrap, global pipe, port, shutdown hooks | [02](02-project-setup-and-structure.md) |
| `app.module.ts` | Root module, imports, middleware configuration | [04](04-modules.md), [12](12-middleware.md) |
| `app.controller.ts` / `app.service.ts` | Basic controller/service pair | [05](05-controllers.md), [06](06-services-and-providers.md) |
| `category/` | Full feature module, controller delegating to service | [04](04-modules.md), [06](06-services-and-providers.md) |
| `customer/` | Feature module, DTO with `class-validator`, interface, in-memory storage | [08](08-dto-and-interfaces.md), [07](07-dependency-injection.md) |
| `employee/` | Minimal module, and an example of a controller *not* using its service (a pattern to avoid) | [05](05-controllers.md) |
| `student/` | Complete REST CRUD example, all five HTTP verbs, `NotFoundException` | [09](09-rest-api-and-http-methods.md) |
| `product/` | `AuthGuard` in use, service and controller with basic unit tests | [11](11-guards-and-authorization.md), [17](17-testing-in-nestjs.md) |
| `user/`, `myname/` | Simple controllers; `myname` also shows a custom pipe on a single field | [10](10-validation-and-pipes.md) |
| `user-roles/` | `RolesGuard` plus a custom `@Roles()` decorator in use | [11](11-guards-and-authorization.md) |
| `guards/auth/` | A basic token-checking `CanActivate` guard | [11](11-guards-and-authorization.md) |
| `guards/roles/` | Role enum, `SetMetadata`-based custom decorator, `Reflector`-based guard | [11](11-guards-and-authorization.md) |
| `middleware/logger/` | A `NestMiddleware` implementation | [12](12-middleware.md) |
| `filters/http-exception/` | A custom `ExceptionFilter` | [13](13-exception-filters.md) |
| `exception/` | Controller using the custom filter plus `ParseIntPipe` | [13](13-exception-filters.md), [10](10-validation-and-pipes.md) |
| `common/pipes/uppercase/` | A custom `PipeTransform` implementation | [10](10-validation-and-pipes.md) |
| `database/` | `OnModuleInit` / `OnApplicationShutdown` lifecycle hooks | [14](14-lifecycle-events.md) |
| `ev/` | `ConfigService` reading an environment variable | [15](15-environment-variables-and-config.md) |
| `*.spec.ts` files throughout `src` | Unit tests | [17](17-testing-in-nestjs.md) |
| `test/app.e2e-spec.ts` | End to end test | [17](17-testing-in-nestjs.md) |

## Where to go from here

At this point you have a complete map of NestJS's core building blocks, grounded entirely in a real, working codebase. The most useful next step is not reading more, it is breaking things on purpose. Try adding a `DELETE` route to `CustomerController` and its matching method in `CustomerService`. Try adding a `Roles(Role.User)` restricted route to `UserRolesController`. Try writing your own custom pipe that rejects negative numbers. Try adding an actual `.env` file with a `DATABASE_URL` value and confirming `GET /ev` returns it. Every one of those exercises reuses a pattern that already exists somewhere in this same codebase, which is exactly why this project is structured as many small, repeated examples rather than one large app: the pattern is what you are really here to learn, not any single feature.
