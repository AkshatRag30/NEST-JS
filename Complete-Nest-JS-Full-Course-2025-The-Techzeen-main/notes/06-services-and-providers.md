# 6. Services and Providers

## What a service is

A service is a plain TypeScript class whose job is to hold logic: fetching data, running calculations, applying rules, talking to a database. The slide deck in `slides/Services in Nest JS.pdf` defines it exactly this way: "a TypeScript class with logic like calculations, data access, etc, used to write business logic in a clean and reusable way."

Crucially, a service does not know it is being used inside a web application. It has no idea what an HTTP request or response is. Look at `src/product/product.service.ts`:

```ts
@Injectable()
export class ProductService {
    private products = [
        { id: 1, name: "Mobile", price: 20000 },
        { id: 2, name: "Tablet", price: 40000 },
        { id: 3, name: "Laptop", price: 80000 },
    ];
    getAllProducts(){
        return this.products;
    }
    getProductById(id: number){
        return this.products.find((product) => product.id === id)
    }
}
```

There is nothing in this file that mentions routes, headers, status codes, or JSON. It is just a class with an array and two plain methods. That is exactly the point. This same class could, in theory, be reused inside a completely different kind of application, a command line tool, a scheduled background job, a GraphQL resolver, without changing a single line, because it has zero dependency on how it gets called.

## The `@Injectable()` decorator

```ts
@Injectable()
export class ProductService { ... }
```

`@Injectable()` marks a class as something NestJS's dependency injection system is allowed to manage. It does two things in practice: it lets this class be injected into other classes (like a controller), and it lets this class itself receive injected dependencies in its own constructor.

Every service in this repo has this decorator: `CategoryService`, `CustomerService`, `EmployeeService`, `StudentService`, `ProductService`, `DatabaseService`, `EvService`. Notice `EmployeeService` is almost empty:

```ts
import { Injectable } from '@nestjs/common';

@Injectable()
export class EmployeeService {}
```

Even with a completely empty body, the decorator is still required if you ever intend to inject this class somewhere. Without `@Injectable()`, NestJS's dependency injection container has no way of knowing this class is meant to participate in the system at all, and trying to inject it elsewhere would fail at startup.

## "Provider" is the bigger category that "service" belongs to

The architecture note ([03-architecture-overview.md](03-architecture-overview.md)) already introduced this distinction, but it is worth repeating precisely, because interviewers and documentation use "provider" constantly. A provider is any class registered in a module's `providers` array that NestJS's dependency injection container knows how to create and hand out. A service is simply a provider whose specific purpose is holding business logic. You could have other kinds of providers too, for example a class purely responsible for talking to an external payment API, or a class that just formats dates, that you might not call a "service" by name but that is architecturally identical, it is registered the same way, injected the same way, and created the same way.

In `src/app.module.ts`:

```ts
providers: [AppService, ProductService, DatabaseService, EvService],
```

All four of these are providers. All four happen to also be services in the conventional naming sense.

## Why keeping logic in services (and out of controllers) actually matters

This is not just a style preference, it produces four concrete benefits, matching the "Why We Use Services" slide:

Testability: because `ProductService` has no dependency on HTTP, you can write a unit test that creates a `ProductService` directly, calls `getProductById(1)`, and checks the result, with zero need to spin up a fake HTTP request. Look at `src/product/product.service.spec.ts` in this repo for exactly this pattern of testing a service in isolation (more in [17-testing-in-nestjs.md](17-testing-in-nestjs.md)).

Reusability: the same service can be called from multiple controllers, or even from another service, without duplicating the logic.

Separation of concerns: routing concerns (what URL, what HTTP verb, how to read parameters) live in one place, and business concerns (what the data actually is and how it should be processed) live in another. When a bug shows up, you immediately know which file to look in based on what kind of bug it is.

Scalability of the codebase: as an app grows, controllers stay small and predictable (every method is roughly three lines: extract input, call service, return output), while all the complexity accumulates in a well organized layer of services that can each be reasoned about independently.

## Injecting a service into a controller, the constructor pattern

Every properly built controller in this repo follows this exact shape, seen here in `src/category/category.controller.ts`:

```ts
@Controller('category')
export class CategoryController {
    constructor(private readonly categoryService: CategoryService){}

    @Get()
    getAllCategories(){
        return this.categoryService.getCategories();
    }
}
```

`constructor(private readonly categoryService: CategoryService)` is doing two things simultaneously, which is a TypeScript shorthand worth understanding fully rather than memorizing. Normally in TypeScript you would write a class property and a constructor separately:

```ts
export class CategoryController {
    private readonly categoryService: CategoryService;
    constructor(categoryService: CategoryService) {
        this.categoryService = categoryService;
    }
}
```

Writing `private readonly categoryService: CategoryService` directly inside the constructor's parameter list is shorthand that does exactly the same thing: it declares the property and assigns it from the constructor argument automatically, in one line. `private` means this property can only be accessed from inside this class. `readonly` means once it is set in the constructor, it can never be reassigned. This pattern is called constructor injection, and it is how virtually all dependency injection happens in NestJS. It is the entire subject of the next note.

## Key terms recap

Service: a class holding business logic, unaware of HTTP.

Provider: the general term for any class NestJS's dependency injection container can create and supply, of which a service is one kind.

`@Injectable()`: the decorator that opts a class into the dependency injection system.

Constructor injection: declaring a dependency as a typed constructor parameter so NestJS supplies an instance of it automatically.
