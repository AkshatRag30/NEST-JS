# 7. Dependency Injection (DI)

## The problem DI solves, shown without NestJS first

Imagine you did not have any framework at all, and you wrote this by hand:

```ts
class CategoryController {
    private categoryService: CategoryService;
    constructor() {
        this.categoryService = new CategoryService();
    }
}
```

This looks harmless for one class. But now imagine `CategoryService` itself needed a database connection class, which needed a configuration class, which needed to read a file from disk. `CategoryController` would now be responsible for knowing how to construct all of that, just to get a `CategoryService` it can use. Every class in your app would be tangled together, tightly coupled to the exact construction details of every class it depends on. Testing becomes painful too: if you want to test `CategoryController` in isolation, you cannot easily swap in a fake `CategoryService`, because the controller is hardcoded to build a real one itself.

## What Dependency Injection actually is

The slide deck defines it precisely: "It is a mechanism where the framework automatically provides the required dependencies, without creating them manually." In NestJS, you never write `new CategoryService()`. Instead, you declare what you need as a typed constructor parameter, and the framework figures out how to build it and hands it to you:

```ts
constructor(private readonly categoryService: CategoryService){}
```

This is called Inversion of Control (IoC): instead of your class controlling when and how its dependencies are created, control is inverted, the framework creates and owns that responsibility, and your class simply declares what it needs.

## How NestJS actually pulls this off (the mechanism, not just the concept)

This is where `emitDecoratorMetadata` and `experimentalDecorators` from `tsconfig.json` (mentioned in [02-project-setup-and-structure.md](02-project-setup-and-structure.md)) come back into play. When TypeScript compiles a class with decorators and `emitDecoratorMetadata` turned on, it embeds extra metadata about the types of a class's constructor parameters into the compiled output, using a library called `reflect-metadata` (listed as a dependency in `package.json`). At runtime, NestJS reads that metadata off of `CategoryController`'s constructor, sees that the first parameter has type `CategoryService`, and looks that type up in its internal registry of known providers.

That registry is built during the `NestFactory.create(AppModule)` call in `main.ts`. NestJS walks through `AppModule`, and every module it imports, collecting every class listed in every `providers` array. Because `src/category/category.module.ts` declares:

```ts
@Module({
  controllers: [CategoryController],
  providers: [CategoryService]
})
export class CategoryModule {}
```

NestJS now knows: "if anything asks for `CategoryService`, here is how to build one, and here is which module it belongs to." When `CategoryController` is created and NestJS sees it needs a `CategoryService` in its constructor, it either creates a fresh instance (the first time it is needed) or, more commonly, hands back an instance it already created earlier, depending on the provider's "scope" (explained below).

This is exactly why `CategoryService` had to be listed in `category.module.ts`'s `providers` array, and why `CategoryController` had to be listed in that same module's `controllers` array, in the same module. If `CategoryService` were only declared as a provider somewhere else and never exported to where `CategoryController` lives, NestJS would throw an error at startup saying it cannot resolve the dependency, because it has no record of how to build it in that context.

## Provider scope: why the "singleton" behavior matters

By default, every provider in NestJS is a singleton, meaning exactly one instance of that class is created for the entire lifetime of the application, and every class that injects it gets that exact same shared instance. This is why `src/customer/customer.service.ts` can safely keep state directly on the class:

```ts
@Injectable()
export class CustomerService {
    private customers: Customer[] = [];

    getAllCustomers(): Customer[] {
        return this.customers;
    }

    addCustomer(createCustomerDto: CreateCustomerDto): Customer {
        const newCustomer: Customer = {
            id: Date.now(),
            ...createCustomerDto,
        };
        this.customers.push(newCustomer);
        return newCustomer;
    }
}
```

`this.customers` starts as an empty array. Every request to `POST /customer` pushes a new entry onto it. This only works correctly because it is the same `CustomerService` instance handling every request; if NestJS created a brand new `CustomerService` for every single incoming request, `this.customers` would reset to empty every time and nothing would ever be remembered between requests. This in-memory array is standing in for a real database in this teaching project (see [16-mongodb-and-nosql-intro.md](16-mongodb-and-nosql-intro.md) for what a real persistence layer would look like instead), but understanding that it survives across requests because of singleton scope is an important detail to internalize now, since it will matter the moment you add a real database.

## Constructor injection, one more time, with a slightly more complex example

`src/guards/roles/roles.guard.ts` shows a provider injecting another provider that comes from NestJS core itself, not from your own code:

```ts
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector){}
  ...
}
```

`Reflector` is a utility class NestJS itself provides globally, so you can inject it into your own classes exactly the same way you inject your own services, using the same constructor injection syntax. There is nothing special about `Reflector` from a syntax point of view, it is injected exactly like `CategoryService` was. This is a nice proof that dependency injection in NestJS is one single, consistent mechanism, whether the thing being injected is something you wrote or something the framework itself ships.

Similarly `src/ev/ev.service.ts` injects `ConfigService`, another framework provided class, from the `@nestjs/config` package:

```ts
@Injectable()
export class EvService {
    constructor(private configureService: ConfigService){}

    getDbUrl(){
        return this.configureService.get<string>('DATABASE_URL')
    }
}
```

More on this exact example in [15-environment-variables-and-config.md](15-environment-variables-and-config.md).

## Why this matters for testing

Because a class only ever declares what it needs (a type), rather than how to build it, you can substitute a fake version of that dependency during testing without changing the class under test at all. Look at how `src/product/product.controller.spec.ts` sets things up (explained fully in [17-testing-in-nestjs.md](17-testing-in-nestjs.md)): NestJS's testing utilities let you build a small, isolated version of the dependency injection container just for the test, and swap in mock providers wherever you want, which would be far harder if `ProductController` manually constructed its own `ProductService` internally.

## Key terms recap

Dependency Injection (DI): a class declares what it needs, and an external system supplies it, instead of the class constructing its own dependencies.

Inversion of Control (IoC): the general principle behind DI, control over object creation is inverted away from the object that uses the dependency.

IoC container: the internal registry NestJS builds during startup that knows how to create every provider and controller in your app.

Singleton scope: the default behavior where one shared instance of a provider is reused everywhere it is injected, for the whole life of the application.

`reflect-metadata`: the library that allows TypeScript's decorator metadata (like constructor parameter types) to be read at runtime, which is what makes automatic dependency injection technically possible.
