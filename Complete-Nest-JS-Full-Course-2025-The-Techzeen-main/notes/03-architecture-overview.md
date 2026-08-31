# 3. Architecture Overview

This note is the map you should keep coming back to. Once these six pieces click, everything else in this course is just detail on top of them.

## The restaurant analogy

The `slides/Modules in Nest JS.pdf` deck uses a restaurant picture, and it is genuinely the clearest way to hold this in your head.

A NestJS module is like a restaurant building. Inside that one building you have a waiter and a kitchen. The waiter is the controller: they take your order (the incoming HTTP request) and bring you back your food (the HTTP response), but they never cook anything themselves. The kitchen is the service: it does the actual work (the business logic, the cooking) but it never talks to the customer directly. The waiter and the kitchen only exist because the restaurant (the module) organizes and contains them.

## The six pieces, one at a time

### 1. Client (the user)

The client is whoever or whatever sends the request: a browser, a mobile app, another server, or a tool like Postman. In this repo, if you send a `GET` request to `http://localhost:3000/student`, you are acting as the client. The client triggers everything else by hitting an endpoint, in this example the endpoint is `/student`.

### 2. Controller

The controller receives that incoming HTTP request and decides which piece of logic should handle it. Look at `src/student/student.controller.ts`:

```ts
@Controller('student')
export class StudentController {
    constructor(private readonly studentService: StudentService){};

    @Get()
    getAll(){
        return this.studentService.getAllStudents();
    }
    ...
}
```

`@Controller('student')` says "every route in this class starts with `/student`". `@Get()` on the `getAll` method says "when a `GET` request arrives at `/student`, run this method". Notice the method itself does almost nothing: it immediately calls `this.studentService.getAllStudents()` and returns whatever comes back. The controller is deliberately thin. It is the receptionist, not the worker. Full detail in [05-controllers.md](05-controllers.md).

### 3. Service

The service holds the actual logic: fetching data, running calculations, applying business rules. Look at `src/student/student.service.ts`:

```ts
@Injectable()
export class StudentService {
    private students = [
        { id: 1, name: 'Farzeen', age: 23 },
        { id: 2, name: 'Ali', age: 25 },
    ];

    getAllStudents(){
        return this.students;
    }
    ...
}
```

The service does not know anything about HTTP. It does not know about status codes, headers, or request objects. It just works with plain data. That separation is the entire point: if tomorrow you swap this app from a REST API to a GraphQL API, or to a command line tool, `StudentService` would not need to change at all, only the controller layer around it would. Full detail in [06-services-and-providers.md](06-services-and-providers.md).

### 4. Provider

"Provider" is the general NestJS term for any class that can be registered with, and handed out by, the dependency injection system. Every service is a provider, but not every provider has to be called a "service". You could have a provider called `Logger`, or `EncryptionHelper`, or `DatabaseConnection`, none of which are strictly "business logic" in the traditional sense, but all of which get created once and reused wherever they are injected. In this repo, `CategoryService`, `CustomerService`, `EmployeeService`, `StudentService`, `ProductService`, `DatabaseService`, and `EvService` are all providers.

### 5. Module

A module is the container that groups a set of related controllers and providers together, plus it declares which other modules it needs. Look at `src/category/category.module.ts`:

```ts
@Module({
  controllers: [CategoryController],
  providers: [CategoryService]
})
export class CategoryModule {}
```

This says "the `category` feature consists of exactly one controller and one provider, bundle them together as a unit called `CategoryModule`". Every NestJS app has at least one module, the root module, which in this project is `AppModule` in `src/app.module.ts`. Full detail in [04-modules.md](04-modules.md).

### 6. Dependency Injection (DI)

Dependency Injection is the mechanism that connects controllers to services (and services to other providers) without you ever writing `new SomeService()` by hand. Look again at `StudentController`'s constructor:

```ts
constructor(private readonly studentService: StudentService){};
```

You never see `this.studentService = new StudentService()` anywhere in this file. NestJS reads the type of the constructor parameter (`StudentService`), looks it up in its internal registry, creates it (only once, by default) or reuses an already created instance, and hands it to `StudentController` automatically the moment that controller is created. Full detail in [07-dependency-injection.md](07-dependency-injection.md).

## Decorators, the glue that makes all of this possible

Every single piece above is only recognizable to NestJS because of a decorator. `@Module()`, `@Controller()`, `@Injectable()`, `@Get()`, `@Post()`, `@Body()`, `@Param()` and so on are all decorators. A decorator is a function, written with an `@` symbol directly above a class, method, or property, that attaches metadata to that thing, or in some cases wraps it with extra behavior. NestJS reads all of that metadata during startup (inside `NestFactory.create(AppModule)`, mentioned in the previous note) to figure out how everything should be wired together.

You will see this pattern constantly:

```ts
@Injectable()
export class ProductService { ... }
```

`@Injectable()` does not change what the class does. It just marks the class as something that can participate in dependency injection, meaning it is allowed to be injected into other classes' constructors, and it is allowed to have its own dependencies injected into it.

## Putting the whole picture together

Here is the full path a single request takes, described in words first, then you will see the same path again, expanded with every additional layer (guards, pipes, middleware, filters), in [18-request-lifecycle-recap.md](18-request-lifecycle-recap.md) once you have learned all of those pieces individually.

A client sends an HTTP request, for example `GET /student/1`. NestJS's underlying HTTP server (Express, by default) receives it. Any registered middleware runs first (see [12-middleware.md](12-middleware.md)). NestJS then finds which module owns the matching route, and inside that module, which controller and which method matches `/student/1`. Before the method body actually runs, any guards attached to that route run to decide whether the request is even allowed through (see [11-guards-and-authorization.md](11-guards-and-authorization.md)). Then any pipes transform or validate the incoming data, like turning the string `"1"` from the URL into the number `1` (see [10-validation-and-pipes.md](10-validation-and-pipes.md)). Only after all of that does the controller method itself run, which typically just calls a method on an injected service. The service does the real work and returns a plain value. That value flows back up through the controller, gets turned into an HTTP response, and is sent back to the client. If anything throws an error anywhere along this path, an exception filter catches it and turns it into a clean, consistent error response instead of crashing the server (see [13-exception-filters.md](13-exception-filters.md)).

## Key terms recap

Module: a container grouping related controllers and providers.

Controller: receives HTTP requests and delegates to services.

Service (a kind of provider): holds business logic, has no knowledge of HTTP.

Provider: the general term for any injectable class.

Dependency Injection: NestJS automatically creating and supplying the objects a class needs, instead of that class creating them itself.

Decorator: an `@` prefixed function that attaches metadata or behavior to code, which NestJS reads to know how to wire the app together.
