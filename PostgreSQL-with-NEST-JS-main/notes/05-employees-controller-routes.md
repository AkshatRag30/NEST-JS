# 05. The Employees Controller, Route by Route

`employees.controller.ts` exposes the `EmployeesService` methods from the previous note as real HTTP routes under the base path `/employees`. This note walks through every route in the order it is actually declared in the file, which matters, since route declaration order affects how NestJS matches incoming requests.

## The full controller

```ts
import { Body, Controller, Delete, Get, Param, Post, Put, Query, UseGuards } from '@nestjs/common';
import { EmployeesService } from './employees.service';
import { Employee } from './employees.entity';
import { SupabaseAuthGuard } from 'src/auth/supabase-auth/supabase-auth.guard';

@Controller('employees')
export class EmployeesController {
    constructor(private readonly employeeService: EmployeesService){}

    @Post()
    async createEmployee(@Body() body: Partial<Employee>): Promise<Employee>{
        return this.employeeService.create(body);
    }

    @UseGuards(SupabaseAuthGuard)
    @Get()
    async findAll(): Promise<Employee[]> {
        return this.employeeService.findAll();
    }
    @Get('search')
    async searchEmployees(@Query('name') name?:string,
        @Query('department') department?:string,): Promise<Employee[]>{
        return this.employeeService.search({ name, department })
    }
     @Get(':id')
    async findOne(@Param('id') id: number): Promise<Employee> {
        return this.employeeService.findOne(id);
    }

    @Put(':id')
    async updateEmployee(
        @Param('id') id: number,
        @Body() body: Partial<Employee>,
    ): Promise<Employee>{
        return this.employeeService.update(id, body);
    }

    @Delete(':id')
    async deleteEmployee(@Param('id') id: number): Promise<{ message: string}> {
        return this.employeeService.delete(id);
    }
}
```

## `POST /employees`, no guard

`createEmployee` reads the whole request body as `@Body() body: Partial<Employee>` and passes it straight to `employeeService.create(body)`. There is no `@UseGuards()` on this route, and there is no DTO class with validation decorators, `Partial<Employee>` is only a TypeScript compile time shape, it places no runtime restriction on what a client can actually send in the request body, someone could `POST /employees` with an empty object, or with extra fields that are not columns on `Employee` at all, and nothing in this route would stop them, TypeORM's `.create()` would simply ignore fields that are not real columns.

## `GET /employees`, the one guarded route

```ts
@UseGuards(SupabaseAuthGuard)
@Get()
async findAll(): Promise<Employee[]> {
    return this.employeeService.findAll();
}
```

This is the only route anywhere in this entire codebase that has `@UseGuards(SupabaseAuthGuard)` attached to it, confirmed by searching the whole `src` folder for every use of `UseGuards` and `SupabaseAuthGuard`. Listing every employee requires a valid Supabase issued JWT in the `Authorization` header, everything else in this controller, and every route in the whole app besides this one, is open to anyone. The full mechanics of what the guard actually checks, and exactly what this means as a security gap, are covered in [07-supabase-auth-guard.md](07-supabase-auth-guard.md).

## `GET /employees/search`, no guard

```ts
@Get('search')
async searchEmployees(@Query('name') name?:string,
    @Query('department') department?:string,): Promise<Employee[]>{
    return this.employeeService.search({ name, department })
}
```

`@Query('name')` and `@Query('department')` read optional query string parameters, so `GET /employees/search?name=an&department=Engineering` calls `employeeService.search({ name: 'an', department: 'Engineering' })`, exercising the `ILIKE` and exact match filtering covered in the previous note. This route is declared before `@Get(':id')` right below it, which is the correct order, since Nest tries routes in the order they are registered, if `:id` had been declared first, a request to `/employees/search` would have matched `findOne` instead, with `id` literally set to the string `"search"`. As written, that mistake does not happen here, `search` is checked first.

## `GET /employees/:id`, no guard, and an unvalidated route parameter

```ts
@Get(':id')
async findOne(@Param('id') id: number): Promise<Employee> {
    return this.employeeService.findOne(id);
}
```

This is worth reading carefully rather than taking the type annotation at face value. `@Param('id')` reads the `:id` segment straight out of the URL, and every URL segment arrives as a plain string, Express and Nest never automatically convert it to a number just because the parameter is typed `id: number`. The `: number` here is only a TypeScript compile time annotation, it does not perform any runtime conversion or validation, there is no `ParseIntPipe` anywhere in this controller (the built in pipe that NestJS provides specifically to convert and validate a route parameter as a real number, throwing a clean `400 Bad Request` if it is not one). In practice this means a request like `GET /employees/abc` would pass straight through into `employeeService.findOne("abc")`, `id` really is the string `"abc"` at runtime despite what the type signature claims, and it would be handed directly to `this.employeeRepository.findOneBy({ id })` in the service. Whether that produces a clean "not found" or an actual database level error depends on how TypeORM and Postgres handle comparing a non numeric string against an integer column, but either way, this is an unvalidated input the same way a missing `ValidationPipe` would be, and it is the same category of gap worth watching for in any NestJS route that takes an id from the URL.

The same missing `ParseIntPipe` gap applies identically to the `:id` parameters in `updateEmployee` and `deleteEmployee` below.

## `PUT /employees/:id`, no guard

`updateEmployee` reads both the `:id` route parameter and a request body typed `Partial<Employee>`, and passes both straight to `employeeService.update(id, body)`, covered in the previous note. Using `PUT` here (rather than `PATCH`) for what is actually a partial merge update (`Object.assign` in the service, not a full replace) is worth noticing if you compare this against strict REST convention, `PUT` traditionally implies replacing the entire resource, while `PATCH` is the verb meant for partial updates, this controller uses `PUT` for a `PATCH` shaped operation.

## `DELETE /employees/:id`, no guard

`deleteEmployee` reads the `:id` parameter and calls `employeeService.delete(id)`, returning the small `{ message: string }` object built in the service.

## The overall picture

Out of five routes on this controller, `create`, `search`, `findOne`, `update`, and `delete` are all reachable by anyone with no authentication at all, and only `findAll` requires a valid token. If the intent behind adding `SupabaseAuthGuard` to this codebase was to protect the employees resource generally, the current code only actually protects the ability to list every employee, every other operation, including creating, searching, reading one specific employee, updating one, and deleting one, remains completely open. Whether that is an intentional demonstration (showing the guard mechanism on one route as a teaching example) or an oversight is not something the code itself states either way, but it is worth reading it exactly as it is rather than assuming the guard protects more than it actually does.
