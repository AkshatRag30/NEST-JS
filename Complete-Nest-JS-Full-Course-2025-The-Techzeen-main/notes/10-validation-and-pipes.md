# 10. Validation and Pipes

## What a pipe is and where it sits in the request flow

A pipe is a class that runs right before a value reaches the parameter of a route handler (a controller method). A pipe can do one of two things: transform the incoming value into a different shape, or validate the incoming value and throw an error if it does not meet some rule (or both at once). The slide deck's phrasing: "A Pipe runs before the data hits the route handler." Pipes are the mechanism that sits between "raw data just arrived over HTTP" and "clean, trustworthy data that the controller method actually receives as an argument."

## Built in pipes: `ParseIntPipe`

The simplest built in pipe used in this repo is `ParseIntPipe`, seen in `src/exception/exception.controller.ts`:

```ts
@Get('hello/:id')
getHello(@Param('id', ParseIntPipe) id: number){
    return {message: `Your ID Is: ${id}`}
}
```

Recall from [05-controllers.md](05-controllers.md) that everything coming out of a URL is a raw string. `@Param('id', ParseIntPipe)` says "take the raw string value of the `id` route parameter, and run it through `ParseIntPipe` before giving it to me." `ParseIntPipe` converts that string into an actual JavaScript number. If the string cannot be converted into a valid integer (say someone requests `/exception/hello/abc`), `ParseIntPipe` throws a `400 Bad Request` error automatically, before `getHello` even runs. This is strictly better than the manual `Number(id)` conversion you saw in `student.controller.ts`, because `Number("abc")` silently produces `NaN` instead of raising any error, which could let broken data slip further into your application unnoticed.

## Building a custom pipe: the `UppercasePipe` example

NestJS lets you write your own pipes for logic specific to your app. `src/common/pipes/uppercase/uppercase.pipe.ts`:

```ts
import { ArgumentMetadata, Injectable, PipeTransform } from '@nestjs/common';

@Injectable()
export class UppercasePipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    if(typeof value === 'string')
    {
      return value.toUpperCase();
    }
    return value;
  }
}
```

Breaking this down: `implements PipeTransform` is a TypeScript interface that requires this class to have a `transform` method with a specific signature. This is how NestJS recognizes something as a valid pipe, structurally, rather than through a decorator. `transform(value: any, metadata: ArgumentMetadata)` receives two things: `value`, the actual incoming data for this parameter, and `metadata`, information about what kind of parameter this is (its expected type, whether it came from `@Body`, `@Param`, `@Query`, and so on), which this particular pipe does not need and therefore does not use. The body of the method is simple: if the value is a string, return an uppercased version of it; otherwise, return the value completely unchanged. Whatever `transform` returns becomes the actual value the controller method receives.

This pipe is used in `src/myname/myname.controller.ts`:

```ts
@Post('custom')
transformName(@Body('name', new UppercasePipe()) name: string){
    return {message: `Received name: ${name}`}
}
```

Notice `new UppercasePipe()`, an actual instance, rather than just the class reference the way `ParseIntPipe` was passed above. Both styles are valid in NestJS. Passing the class reference (`ParseIntPipe`) lets NestJS instantiate it for you using dependency injection, which matters more if the pipe itself needs injected dependencies. Passing an already created instance (`new UppercasePipe()`) works fine for a pipe with no dependencies of its own, like this one. If a client sends `{"name": "zara"}` to `POST /myname/custom`, the response will be `{"message": "Received name: ZARA"}`, because the pipe transformed the value before the method body ever ran.

## `ValidationPipe`: the pipe that actually enforces your DTOs

This is the most important pipe in the whole project, and it is registered globally in `main.ts`:

```ts
app.useGlobalPipes(new ValidationPipe({
  whitelist: true,
  forbidNonWhitelisted: true
}))
```

`ValidationPipe` is a pipe built into `@nestjs/common` specifically to work together with `class-validator` decorators on your DTOs (fully explained in [08-dto-and-interfaces.md](08-dto-and-interfaces.md)). Because it is registered with `useGlobalPipes`, it automatically applies to every single route in the entire application, on every parameter that has a DTO class as its type, without you needing to attach anything manually per route.

`whitelist: true` means: for any property present on the incoming object that is not declared with a decorator in the target DTO class, strip it out silently before validation even happens.

`forbidNonWhitelisted: true` changes what happens to those same stray, undeclared properties: instead of quietly removing them, it makes their mere presence an error, and the whole request gets rejected with a `400 Bad Request`. Using both options together (as this project does) means: any field you did not explicitly declare on your DTO is treated as a mistake by the client, not silently tolerated.

Walking through `CreateCustomerDto` (`src/customer/dto/create-customer.dto.ts`) one more time with this lens:

```ts
export class CreateCustomerDto {
  @IsString()
  name: string;
  @IsInt()
  age: number;
}
```

If a `POST /customer` request body is `{"name": "Bilal", "age": 30}`, it passes cleanly. If it is `{"name": "Bilal", "age": "thirty"}`, `@IsInt()` fails because `"thirty"` is a string, not an integer, and `ValidationPipe` rejects the request. If it is `{"name": "Bilal", "age": 30, "extra": "hack"}`, `forbidNonWhitelisted` rejects the request because of the undeclared `extra` field.

## Where you can apply a pipe: three levels of scope

This mirrors the same "three levels" pattern you already saw with filters and guards in [05-controllers.md](05-controllers.md).

Global scope: `app.useGlobalPipes(...)` in `main.ts`, applies to every route in the whole app. This project's `ValidationPipe` is set up this way.

Parameter scope: passed directly as a second argument to a decorator like `@Param('id', ParseIntPipe)` or `@Body('name', new UppercasePipe())`, applies to just that one argument of just that one method. Both examples used in this repo are set up this way.

There is also a method level and controller level scope, using the `@UsePipes(...)` decorator above a method or class, which this repo does not happen to use, but which works the same way `@UseFilters(...)` and `@UseGuards(...)` do (see [11-guards-and-authorization.md](11-guards-and-authorization.md)).

## Key terms recap

Pipe: a class that transforms and/or validates a value before it reaches a route handler's parameter.

`PipeTransform`: the interface a custom pipe must implement, requiring a `transform(value, metadata)` method.

`ValidationPipe`: NestJS's built in pipe that validates an incoming object against a DTO class's `class-validator` decorators.

`whitelist` / `forbidNonWhitelisted`: `ValidationPipe` options controlling what happens to properties on the incoming data that were not declared on the DTO.

`ParseIntPipe`: a built in pipe that converts a string (commonly from a route parameter) into a number, throwing an error if the conversion is not possible.
