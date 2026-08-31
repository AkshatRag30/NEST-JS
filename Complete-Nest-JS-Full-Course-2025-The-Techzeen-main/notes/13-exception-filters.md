# 13. Exception Filters

## The problem exception filters solve

Without any special handling, if a piece of code inside a controller or service throws an error, the framework has to decide, on its own, what HTTP response to send back. NestJS actually already does something reasonable by default (it turns thrown `HttpException`s into a matching JSON error response), but you often want to control that shape yourself, for example to guarantee every single error response from your API has the exact same consistent structure (the same fields, in the same order, formatted the same way), no matter which part of the app threw the error. That is exactly what an exception filter is for: handling errors and exceptions in one centralized, consistent place, instead of scattering try/catch blocks with ad hoc response formatting all over your controllers.

## Built in exceptions you have already seen

Before writing a custom filter, it is worth being clear on what you are actually catching. NestJS ships a set of built in exception classes, all extending a base `HttpException` class, each corresponding to a standard HTTP status code. You already saw one in `src/student/student.service.ts`:

```ts
if(!student) throw new NotFoundException('Student not Found!');
```

`NotFoundException` automatically results in a response with status code `404` and a body containing the message you passed in. Other common ones include `BadRequestException` (`400`), `UnauthorizedException` (`401`), `ForbiddenException` (`403`), and plain `HttpException` itself, which lets you specify any status code directly. All of these are "just" ways of throwing an error that NestJS already knows how to translate into a proper HTTP response, without any custom filter needed at all. A custom filter becomes useful when you want to change what that resulting response actually looks like.

## Building a custom filter: `HttpExceptionFilter`

`src/filters/http-exception/http-exception.filter.ts`:

```ts
import { ArgumentsHost, Catch, ExceptionFilter, HttpException } from '@nestjs/common';
import { Request, Response } from 'express';

@Catch()
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();

    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
      message: exception.message
    })
  }
}
```

`@Catch()` is the decorator that marks this class as an exception filter, and it also declares which type of exception it should handle. Called with no arguments, as it is here, it catches every exception thrown anywhere in scope, regardless of type. You can instead write `@Catch(NotFoundException)` to have a filter that only reacts to that one specific exception type, letting every other kind of error fall through to NestJS's default handling (or to another filter).

`implements ExceptionFilter` requires this class to have a `catch(exception, host)` method. `exception` is the actual error object that was thrown. `host: ArgumentsHost` is very similar in spirit to the `ExecutionContext` you saw in guards, it is a protocol-agnostic wrapper around the current request context, and `host.switchToHttp()` narrows it down to the HTTP-specific case, giving you access to the real `request` and `response` objects.

`exception.getStatus()` reads the HTTP status code off the exception (this method exists because `HttpException`, and everything that extends it like `NotFoundException`, carries a status code internally). `response.status(status).json({...})` then manually builds and sends the final JSON response, giving you complete control over its shape: `statusCode`, a `timestamp` (freshly generated at the moment of the error), the `path` the client actually requested, and the exception's own `message`. Every single error this filter handles, no matter where it originated in your app, will come back to the client in this exact same shape, which is a huge usability win for whoever is building a frontend against this API, since they only ever need to write error handling code for one consistent error format.

## Applying the filter to a controller

`src/exception/exception.controller.ts`:

```ts
@Controller('exception')
@UseFilters(HttpExceptionFilter)
export class ExceptionController {
    @Get('hello/:id')
    getHello(@Param('id', ParseIntPipe) id: number){
        return {message: `Your ID Is: ${id}`}
    }
}
```

`@UseFilters(HttpExceptionFilter)` above the class attaches the filter to every route inside `ExceptionController`. If a client requests `GET /exception/hello/abc`, `ParseIntPipe` (see [10-validation-and-pipes.md](10-validation-and-pipes.md)) throws an error because `"abc"` cannot be converted to a number, and `HttpExceptionFilter` catches that error and formats the resulting response using the shape defined above, instead of whatever NestJS's own default error formatting would have produced.

## The three scopes filters can be applied at

This is the exact same pattern you have now seen repeated for guards and pipes:

Method scope: `@UseFilters(HttpExceptionFilter)` directly above one route handler method, affecting only that route.

Controller scope: `@UseFilters(HttpExceptionFilter)` directly above the class, as shown here, affecting every route in that controller.

Global scope: registered inside `main.ts` with `app.useGlobalFilters(new HttpExceptionFilter())`, affecting every route in the entire application. This repo does not currently do this (the filter is only applied at the controller level, to `ExceptionController`), but it is the natural next step if you wanted every error across the whole app to share this exact response shape, and it is exactly analogous to how `ValidationPipe` is registered globally in `main.ts` in this same project.

## Key terms recap

Exception filter: a class that intercepts errors thrown during request handling and controls exactly what response is sent back.

`@Catch()`: the decorator marking a class as an exception filter, optionally scoped to one specific exception type.

`ExceptionFilter`: the interface an exception filter implements, requiring a `catch(exception, host)` method.

`ArgumentsHost`: the protocol-agnostic object passed to a filter's `catch` method, used to get the real request/response objects.

`HttpException` (and its subclasses like `NotFoundException`): NestJS's built in error classes that carry an HTTP status code and a message.
