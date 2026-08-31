# 05. CRUD with Mongoose Models

The `student` module is the only module in this repo with a full create, read, update, and delete API, and it is also the only one wired up to real HTTP route parameters and request bodies instead of hardcoded demo data. This note walks through `student.service.ts` and `student.controller.ts` method by method.

## The controller: routes and where data comes from

```ts
// src/student/student.controller.ts
@Controller('student')
export class StudentController {
    constructor(private readonly studentService: StudentService){}

    @Post()
    async addStudent(@Body() data: Partial<Student>){
        return this.studentService.createStudent(data);
    }
    @Get()
    async getStudents(){
        return this.studentService.getAllStudents();
    }
    @Get(':id')
     async getStudent(@Param('id') id: string){
        return this.studentService.getStudentById(id);
    }
    @Put(':id')
    async updateStudent(
        @Param('id') id: string,
        @Body() data: Partial<Student>,
    ){
        return this.studentService.updateStudent(id,data);
    }
    @Patch(':id')
    async patchStudent(
        @Param('id') id: string,
        @Body() data: Partial<Student>,
    ){
        return this.studentService.patchStudent(id,data);
    }

    @Delete(':id')
    async deleteStudent(@Param('id') id: string) {
        return this.studentService.deleteStudent(id);
    }
}
```

`@Controller('student')` means every route in this class is prefixed with `/student`. Each method decorator (`@Post()`, `@Get()`, `@Get(':id')`, `@Put(':id')`, `@Patch(':id')`, `@Delete(':id')`) maps an HTTP verb and path onto a method, exactly the standard NestJS routing pattern. `@Body()` pulls the parsed JSON request body into the `data` parameter, and `@Param('id')` pulls the `:id` segment out of the URL path. Every one of these controller methods is a thin pass through, its whole job is to read data off the request and hand it to the matching `StudentService` method, which is where the actual Mongoose work happens.

The type `Partial<Student>` used for the request body is TypeScript's built in utility type meaning "every property of `Student`, but all of them optional." This is a shortcut standing in for a proper DTO class with `class-validator` decorators (the kind of pattern you may have seen in other NestJS projects); here, it means the controller will happily accept a request body missing `name` or `age` and pass it straight through to Mongoose, letting Mongoose's own `required: true` validation (see [03-schemas-and-props.md](03-schemas-and-props.md)) be the only thing standing between bad input and a saved document.

## Create: `createStudent`

```ts
async createStudent(data: Partial<Student>): Promise<Student> {
    const newStudent = new this.studentModel(data);
    return newStudent.save();
}
```

`this.studentModel` is the injected Mongoose `Model` from the previous note. Calling `new this.studentModel(data)` constructs a new, unsaved document in memory, populated with whatever fields `data` provided; nothing has touched the database yet at this point. Calling `.save()` on that instance is the step that actually sends an insert command to MongoDB, running Mongoose's schema validation (the `required: true` rules) first, and returns a promise that resolves with the saved document, now including its generated `_id` and, thanks to `{ timestamps: true }` on the schema, its `createdAt`/`updatedAt` fields.

## Read all: `getAllStudents`

```ts
async getAllStudents(): Promise<Student[]> {
    return this.studentModel.find().exec();
}
```

`this.studentModel.find()` with no arguments builds a query that matches every document in the `students` collection. Calling `.exec()` actually sends that query and returns a real promise. Technically, Mongoose query objects are already "thenable" (you can `await` them directly without calling `.exec()`), but calling `.exec()` explicitly is a common convention in Mongoose code because it makes it unambiguous to a reader that this line is a database call returning a genuine `Promise`, rather than some other thenable-like object; you will see this same `.exec()` pattern on every query method in this service.

## Read one: `getStudentById`

```ts
async getStudentById(id: string): Promise<Student | null> {
    return this.studentModel.findById(id).exec();
}
```

`findById` is Mongoose's shorthand for `findOne({ _id: id })`. Note the return type is explicitly `Promise<Student | null>`, because if no document exists with that `_id`, MongoDB returns nothing and Mongoose resolves with `null` rather than throwing an error, this is why `strictNullChecks` (see [01-project-setup-and-bootstrap.md](01-project-setup-and-bootstrap.md)) matters, TypeScript forces this possibility to be reflected in the type rather than silently assuming a student is always found.

## Update, the interesting one: `updateStudent` versus `patchStudent`

This is the part of the module worth slowing down on, because the code deliberately implements `PUT` and `PATCH` with two different Mongoose update strategies, matching what those two HTTP verbs are supposed to mean.

```ts
async updateStudent(id: string, data: Partial<Student>): Promise<Student | null> {
    // return this.studentModel.findByIdAndUpdate(id, data, {new: true}).exec();
    const updated = await this.studentModel.findByIdAndUpdate(id, {
        name: data.name ?? null,
        age: data.age ?? null,
        email: data.email ?? null,
    }, { overwrite: true, new: true});
    return updated;
}

async patchStudent(id: string, data: Partial<Student>): Promise<Student | null>{
    return this.studentModel.findByIdAndUpdate(id, data, {new: true}).exec();
}
```

`patchStudent`, wired to the `@Patch(':id')` route, is the simpler of the two: `findByIdAndUpdate(id, data, { new: true })` merges whatever fields `data` contains into the existing document, leaving every field not present in `data` completely untouched. `{ new: true }` tells Mongoose to return the document as it looks after the update, rather than its default behavior of returning the document as it looked before the update. This is exactly the semantics `PATCH` is supposed to have in REST, a partial update.

`updateStudent`, wired to the `@Put(':id')` route, is written to behave like a full replacement instead, matching what `PUT` is supposed to mean in REST: the entire resource gets replaced by whatever the client sent. The commented out line directly above it, `findByIdAndUpdate(id, data, {new: true})`, is what the author originally tried, and it is functionally identical to `patchStudent`, a merge, not a replace. The active code below it fixes that by first building a brand new plain object with exactly three keys, `name`, `age`, and `email`, explicitly falling back to `null` with `data.name ?? null` for any field the caller did not send, and then passing `{ overwrite: true }` as an option. `overwrite: true` is the specific Mongoose option that changes `findByIdAndUpdate`'s behavior from "merge these fields into the existing document" to "replace the entire document body with exactly what I am giving you," which means any field on the existing document that is not one of `name`, `age`, or `email` in the new object would be wiped out, and any of those three fields the caller omitted gets explicitly set to `null` rather than being left as its old value. This is the correct way to implement true `PUT` replace semantics with Mongoose, and the comment left in the code is a small, honest trace of the author debugging their way from the "merge" version to the "replace" version.

## Delete: `deleteStudent`

```ts
async deleteStudent(id: string): Promise<Student | null> {
    return this.studentModel.findByIdAndDelete(id).exec();
}
```

`findByIdAndDelete` removes the matching document and resolves with the document as it looked immediately before deletion, or `null` if no document with that `id` existed. There is nothing unusual here, this is the last of the five REST operations (create, read all, read one, update, delete) that together make this the only module in the repo with a complete CRUD surface.
