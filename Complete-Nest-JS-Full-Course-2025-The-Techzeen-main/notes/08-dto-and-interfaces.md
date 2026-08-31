# 8. DTOs and Interfaces

This note covers two closely related but distinct TypeScript concepts that this repo uses together: interfaces and DTOs (Data Transfer Objects).

## Interfaces: describing a shape, at compile time only

A TypeScript interface describes what properties an object must have, and their types. It exists purely for the TypeScript compiler, and it disappears entirely once your code is compiled to JavaScript. Look at `src/customer/interfaces/customer.interface.ts`:

```ts
export interface Customer{
    id: number;
    name: string;
    age: number;
}
```

This says "anything typed as `Customer` must have a numeric `id`, a string `name`, and a numeric `age`, nothing more is required or promised by the type itself." It is used in `src/customer/customer.service.ts`:

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

`private customers: Customer[] = [];` tells TypeScript "this array will only ever contain objects shaped like `Customer`." If somewhere else in the code you accidentally tried to push an object missing `age`, TypeScript would refuse to compile. This is exactly what the slide deck means by "type safe code": mistakes about the shape of your data are caught while you are writing the code, not later when a user hits a broken endpoint in production.

Because an interface is compile time only, you cannot use it to check data that is coming from outside your program (like an actual HTTP request body) at runtime, because by the time your code is actually running, the interface no longer exists in any form the running program can inspect. This is precisely the gap DTOs fill.

## DTOs: describing a shape that also exists at runtime

A DTO (Data Transfer Object) is an object whose job is to carry data between layers, most commonly between the client and your backend. In this repo, DTOs are written as classes, not interfaces, and that difference is deliberate and important. Look at `src/customer/dto/create-customer.dto.ts`:

```ts
import { IsInt, IsString } from 'class-validator';

export class CreateCustomerDto {
  @IsString()
  name: string;
  @IsInt()
  age: number;
}
```

Because this is a class, not an interface, it still exists as a real, usable thing when your compiled code actually runs, unlike an interface which vanishes at compile time. That matters because `class-validator`'s decorators (`@IsString()`, `@IsInt()`) need something real to attach validation rules to at runtime. This is the whole reason the slide deck says DTOs help "ensure only required data is passed (security and validation)": because a DTO class survives into the running application, NestJS's `ValidationPipe` can actually inspect an incoming request body against it while the app is live, and reject anything that does not match.

## How the DTO gets used and validated, end to end

In `src/customer/customer.controller.ts`:

```ts
@Post()
addCustomer(@Body() createCustomerDto: CreateCustomerDto) {
    return this.customerService.addCustomer(createCustomerDto);
}
```

And back in `main.ts`:

```ts
app.useGlobalPipes(new ValidationPipe({
  whitelist: true,
  forbidNonWhitelisted: true
}))
```

Here is the full sequence of what happens when a client sends `POST /customer` with a JSON body like `{"name": "Aiman", "age": 22}`:

The global `ValidationPipe` intercepts the incoming body before the controller method runs. It looks at the parameter type declared on `addCustomer`, which is `CreateCustomerDto`. It uses `class-transformer` internally to turn the plain incoming JSON object into an actual instance of the `CreateCustomerDto` class. Then it runs `class-validator`'s checks against that instance, based on the decorators you wrote: `@IsString()` on `name` checks that `name` really is a string, `@IsInt()` on `age` checks that `age` really is a whole number. If either check fails, `ValidationPipe` automatically throws a `400 Bad Request` error, and `addCustomer` never even runs. If a client sent `{"name": "Aiman", "age": 22, "isAdmin": true}`, the extra `isAdmin` field would be silently deleted because of `whitelist: true`, or the whole request would be rejected because of `forbidNonWhitelisted: true` (this project uses both settings together, so the outcome here is rejection, since `forbidNonWhitelisted` makes any non-whitelisted property an error rather than something to quietly strip). This connects directly to [10-validation-and-pipes.md](10-validation-and-pipes.md), where `ValidationPipe` itself is explained fully.

Only if validation passes does `addCustomer(createCustomerDto)` run, and by that point, `createCustomerDto` is guaranteed to actually have a string `name` and a numeric `age`, exactly as declared.

## DTO versus Interface, side by side

Both `CreateCustomerDto` and `Customer` describe data shapes, but they serve different moments in the data's life and are written differently on purpose.

`Customer` (the interface) describes the full, final shape of data once it exists inside your system, including fields the system itself generates, like `id`. Notice `Customer` has `id`, but `CreateCustomerDto` does not, because a client creating a new customer should never be the one deciding its `id`, the server generates that (`id: Date.now()` in `CustomerService`). An interface is the right tool here because this shape is only ever used internally, inside TypeScript code, never validated against raw untrusted input.

`CreateCustomerDto` (the DTO, a class) describes exactly what a client is allowed to send in, and nothing more. It needs to be a class, decorated with `class-validator` decorators, precisely because it must survive into the running application to actually validate real incoming data.

A simple rule of thumb for your own future projects: if the data is coming in from outside your app (a request body, query parameters) and needs validation, write a DTO class. If the data only describes the internal shape of something your own code already trusts, a plain interface is enough and is slightly lighter weight since it costs nothing at runtime.

## Interfaces used for responses too

The DTO slide deck also mentions interfaces being used for response objects, not just requests. In this repo, `Customer` plays exactly that dual role: it types the internal storage array (`private customers: Customer[]`) and it types what `getAllCustomers(): Customer[]` and `addCustomer(...): Customer` return. This is a common, sensible pattern: use one interface to represent "the full internal shape of this entity" and reuse it both as your storage type and your return type, while using separate, narrower DTO classes for each kind of incoming request (a `CreateCustomerDto` might look different from an eventual `UpdateCustomerDto`, since an update might allow every field to be optional, similar to how `student.controller.ts` used `Partial<{...}>` for its `PATCH` route, see [05-controllers.md](05-controllers.md)).

## Key terms recap

Interface: a TypeScript-only construct describing an object's shape, checked at compile time, erased at runtime.

DTO (Data Transfer Object): an object, written as a class in NestJS, that defines and (with `class-validator`) enforces the shape of data moving between layers, especially incoming request data.

`class-validator`: the library providing decorators like `@IsString()` and `@IsInt()` used to declare validation rules directly on a DTO's properties.

`class-transformer`: the library NestJS's `ValidationPipe` uses internally to convert a plain incoming object into an actual instance of your DTO class before validating it.
