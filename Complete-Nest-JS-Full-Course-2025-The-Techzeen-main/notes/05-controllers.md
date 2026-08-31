# 5. Controllers

## What a controller's job is, and is not

A controller's only job is to receive an HTTP request and return a response, delegating any real work to a service. It should not contain business logic, calculations, or data storage. If you find yourself writing an `if` statement that checks business rules directly inside a controller method, that logic almost always belongs in a service instead.

## The `@Controller()` decorator

```ts
@Controller('student')
export class StudentController {
```

The string you pass to `@Controller()` is a route prefix. Every method inside this class that has its own HTTP method decorator gets that prefix automatically prepended to its path. So `@Get()` with no argument inside `StudentController` means `GET /student`, and `@Get(':id')` means `GET /student/:id`.

You can also write `@Controller()` with no argument at all, which is exactly what `src/app.controller.ts` does:

```ts
@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get()
  getHello(): string {
    return this.appService.getHello();
  }
}
```

With no prefix, `@Get()` here maps to the literal root path `/`.

## The five HTTP method decorators used in this repo

NestJS gives you one decorator per HTTP verb: `@Get()`, `@Post()`, `@Put()`, `@Patch()`, `@Delete()`. The clearest place to see all five used together is `src/student/student.controller.ts`, which implements a complete CRUD (Create, Read, Update, Delete) API for a list of students:

```ts
@Controller('student')
export class StudentController {
    constructor(private readonly studentService: StudentService){};

    @Get()
    getAll(){
        return this.studentService.getAllStudents();
    }

    @Get(':id')
    getOne(@Param('id') id: string){
        return this.studentService.getStudentById(Number(id))
    }

    @Post()
    create(@Body() body: { name: string; age: number } ){
        return this.studentService.createStudent(body);
    }

    @Put(':id')
    update(@Param('id') id: string, @Body() body: { name: string; age: number} ){
        return this.studentService.updateStudent(Number(id), body);
    }
    @Patch(':id')
    patch(@Param('id') id: string, @Body() body: Partial<{ name: string; age: number}> ){
        return this.studentService.patchStudent(Number(id), body);
    }
    @Delete(':id')
    remove(@Param('id') id:string){
        return this.studentService.deleteStudent(Number(id));
    }

}
```

Go through each route and match it to what it does, and to [09-rest-api-and-http-methods.md](09-rest-api-and-http-methods.md) for what the HTTP verb itself semantically means:

`GET /student` calls `getAll()`, which asks the service for every student.

`GET /student/:id` calls `getOne()`. The `:id` in the decorator is a route parameter placeholder. Whatever the client puts in that position of the URL (for example the `5` in `/student/5`) becomes available through `@Param('id')`.

`POST /student` calls `create()`, reading the new student's data out of the request body with `@Body()`.

`PUT /student/:id` calls `update()`, which is meant to replace the entire student record at that id.

`PATCH /student/:id` calls `patch()`, which is meant to update only some fields, notice the type is `Partial<{ name: string; age: number }>`, meaning every field is optional here, unlike the full `{ name: string; age: number }` type used for `create` and `update`.

`DELETE /student/:id` calls `remove()`, removing that student.

## `@Param()`, decoding route parameters

```ts
@Get(':id')
getOne(@Param('id') id: string){
    return this.studentService.getStudentById(Number(id))
}
```

`:id` inside the route path is a placeholder. `@Param('id')` extracts exactly that named piece from the incoming URL and injects it as the `id` argument of the method. One crucial detail beginners trip over: everything that comes out of a URL is always a string, even if it looks like a number. That is exactly why the code immediately wraps it: `Number(id)`, converting the string `"5"` into the actual number `5` before handing it to the service. You will see in [10-validation-and-pipes.md](10-validation-and-pipes.md) that NestJS also offers a pipe, `ParseIntPipe`, that does this conversion automatically and more safely (it throws a proper error if the value is not actually a valid number, instead of silently producing `NaN`). Compare this manual `Number(id)` approach here to `src/exception/exception.controller.ts`, which uses `ParseIntPipe` instead, to see both styles side by side.

You can also grab all route parameters at once with `@Param()` with no argument, which gives you an object containing every named parameter, but this repo always uses the named form, `@Param('id')`, which is the more explicit and readable style.

## `@Body()`, reading the request payload

```ts
@Post()
create(@Body() body: { name: string; age: number } ){
    return this.studentService.createStudent(body);
}
```

`@Body()` extracts the parsed JSON body of the incoming request. When a client sends a `POST` request with a JSON payload like `{"name": "Zara", "age": 21}`, `@Body()` hands you that entire object. Here the type annotation is written inline (`{ name: string; age: number }`), which works but is considered messy style once a shape gets reused. `src/customer/customer.controller.ts` shows the cleaner alternative, using a dedicated DTO class instead of an inline type:

```ts
@Post()
addCustomer(@Body() createCustomerDto: CreateCustomerDto) {
    return this.customerService.addCustomer(createCustomerDto);
}
```

DTOs are their own topic, fully explained in [08-dto-and-interfaces.md](08-dto-and-interfaces.md), because they do more than just describe a shape, they also enable automatic validation.

You can also pass a specific key to `@Body()`, like `@Body('name')`, to extract just one field instead of the whole object. That exact pattern is used in `src/myname/myname.controller.ts`:

```ts
@Post('custom')
transformName(@Body('name', new UppercasePipe()) name: string){
    return {message: `Received name: ${name}`}
}
```

Here `@Body('name', new UppercasePipe())` pulls out just the `name` field from the request body, and immediately runs it through a custom pipe before it ever reaches the method body. Pipes are covered in [10-validation-and-pipes.md](10-validation-and-pipes.md).

## Controllers that inject a service versus controllers that do not

Compare `src/employee/employee.controller.ts`:

```ts
@Controller('employee')
export class EmployeeController {
    @Get()
        getEmployee(){
            return 'Employee data fetched successfully!!';
        }
}
```

to `src/category/category.controller.ts`:

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

`EmployeeController` never injects `EmployeeService` and just returns a hardcoded string directly. This works, and NestJS does not require a controller to use a service, but it defeats the entire purpose of the architecture described in [03-architecture-overview.md](03-architecture-overview.md). `CategoryController` shows the pattern you should actually follow: the controller stays "thin" and delegates the real answer to `categoryService.getCategories()`. As you build your own features, always aim for the `CategoryController` shape, not the `EmployeeController` shape, even for something this simple, because it is the shape that scales.

## Applying guards, filters, and pipes at the controller level

You will meet these decorators properly in their own notes, but it is worth noticing here that they can be stacked directly above `@Controller()` or above an individual method. For example, `src/exception/exception.controller.ts`:

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

`@UseFilters(HttpExceptionFilter)` above the whole class means every route inside this controller uses that custom error handler. `ParseIntPipe`, passed as a second argument straight into `@Param('id', ParseIntPipe)`, is applied to just that one parameter of just that one method. This shows the general pattern in NestJS: almost every cross cutting feature (guards, pipes, filters, interceptors) can be applied at three different scopes, globally (in `main.ts`), at the controller level (above the class), or at the method/parameter level (right where it is used), and you choose the narrowest scope that fits what you actually need.

## Key terms recap

Route: a combination of an HTTP method and a URL path that maps to one controller method.

Route parameter: a named placeholder in a path (like `:id`) whose value comes from the actual URL the client requested.

`@Param()`: decorator that extracts route parameters.

`@Body()`: decorator that extracts the parsed request body.

CRUD: Create, Read, Update, Delete, the four basic operations most APIs need to support for any resource.
