# 04. forFeature and Dependency Injection

Defining a schema class (covered in the previous note) only describes the shape of data. It does not, by itself, give you anything you can call `.find()` or `.save()` on inside a service. That is the job of two more pieces: `MongooseModule.forFeature()` at the module level, and `@InjectModel()` at the service level. This note walks through both using `src/student/student.module.ts` and `src/student/student.service.ts`.

## Registering a schema with a module: `MongooseModule.forFeature()`

```ts
// src/student/student.module.ts
@Module({
    imports: [
        MongooseModule.forFeature([{ name: Student.name, schema: StudentSchema }])
    ],
    providers: [StudentService],
    controllers: [StudentController]
})
export class StudentModule {}
```

Where `MongooseModule.forRoot()` (covered in [02-connecting-to-mongodb.md](02-connecting-to-mongodb.md)) establishes the one shared connection to MongoDB for the whole app, `MongooseModule.forFeature([...])` is what you call inside each individual feature module to say "within this module, I want to work with documents shaped like this schema."

The array passed to `forFeature` can register more than one schema at once, and several modules in this repo do exactly that, for example `src/employee/employee.module.ts` registers both `Employee` and `Profile` in the same call, since `EmployeeService` needs to talk to both collections. Each entry is an object with two keys: `name`, which is the string MongoDB collection name Mongoose will use internally (by convention this is written as `Student.name`, which is just the class's name as a string, `"Student"`, rather than typing the string by hand, so that if the class is ever renamed, this reference updates automatically), and `schema`, the actual compiled schema object produced by `SchemaFactory.createForClass()` in the previous note.

Behind the scenes, this call is what causes NestJS's dependency injection container to create and register an injectable provider for a Mongoose `Model` object tied to this specific schema, scoped to this module. That provider is identified internally by a special token derived from the string `"Student"` (specifically, something like `getModelToken('Student')`), which is exactly what `@InjectModel(Student.name)` on the other end asks for by name.

## Injecting the model into a service: `@InjectModel()`

```ts
// src/student/student.service.ts
@Injectable()
export class StudentService {
    constructor(
        @InjectModel(Student.name) private studentModel: Model<StudentDocument>
    ) { }
    // ...
}
```

`@Injectable()` marks `StudentService` as a provider that Nest's dependency injection container is allowed to construct and hand to whoever asks for it, exactly the same decorator used on every plain service in any NestJS app.

`@InjectModel(Student.name)` is a parameter decorator, and it is the piece that is specific to `@nestjs/mongoose`. Normally, NestJS figures out what to inject into a constructor parameter purely from its TypeScript type (this is how a plain service gets another plain service injected, just by naming its class as the parameter type). That trick does not work for Mongoose models, because `Model<StudentDocument>` is a generic type from the `mongoose` library itself, not a class this project defines, so there is no unique class for Nest to match against. `@InjectModel(Student.name)` solves that ambiguity explicitly, by telling Nest "for this specific parameter, look up the provider that was registered under the token for `'Student'`", which is precisely the provider that `MongooseModule.forFeature([{ name: Student.name, ... }])` created back in the module.

The parameter's TypeScript type, `Model<StudentDocument>`, is what then gives you compile time type safety and autocomplete on every Mongoose method you call on `this.studentModel` throughout the rest of the class, `Model` is a generic type from `mongoose` representing "a queryable, saveable collection of documents shaped like `StudentDocument`."

## Why this matters: constructor injection, the same idea as anywhere else in NestJS

If you have already learned plain NestJS dependency injection (a controller receiving a service in its constructor, for example), this whole mechanism is exactly that same pattern, just applied to a different kind of provider. `MongooseModule.forFeature(...)` is essentially a shorthand for registering a `Model` object as a provider in this module's `providers` array, the way you might otherwise write a custom `provide`/`useFactory` pair yourself, and `@InjectModel(...)` is essentially a shorthand for `@Inject(SOME_TOKEN)` where `SOME_TOKEN` happens to be the specific token Mongoose integration uses internally. Once you see it that way, none of this is really new machinery, it is the exact same NestJS inversion of control container you already rely on for every other provider in this codebase, just pointed at a Mongoose model instead of a class you wrote yourself.
