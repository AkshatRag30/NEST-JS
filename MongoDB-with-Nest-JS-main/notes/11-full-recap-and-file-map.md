# 11. Full Recap: One Request, Start to Finish

This last note traces one complete HTTP request through every layer covered in this folder, and then gives you a single table mapping every file in `src` to the note that explains it, so this folder can double as a quick reference after your first read through.

## Tracing `POST /library`, from HTTP request to MongoDB and back

Picking `library.controller.ts`'s `createLibrary()` route, since it touches almost every concept in this folder at once:

A client sends `POST /library` with no body needed, since `createLibrary()` takes no parameters. Nest's router, built when `NestFactory.create(AppModule)` ran at startup (see [01-project-setup-and-bootstrap.md](01-project-setup-and-bootstrap.md)), matches this request to `LibraryController.createLibrary`, because `@Controller('library')` combined with `@Post()` on that method is exactly `POST /library`.

`LibraryController`'s constructor already has a working `LibraryService` instance in hand, injected automatically back when the app started, thanks to `LibraryModule` listing `LibraryService` in its `providers` array. The controller method does nothing but delegate: `return this.libraryService.createLibrary();`.

Inside `LibraryService.createLibrary()`, `this.bookModel` and `this.libraryModel`, both injected via `@InjectModel(...)` (see [04-forfeature-and-dependency-injection.md](04-forfeature-and-dependency-injection.md)), are Mongoose `Model` objects tied to the `books` and `libraries` collections respectively, made possible because `LibraryModule` registered both schemas with `MongooseModule.forFeature([...])`, and both of those schemas were themselves defined earlier using `@Schema()`/`@Prop()`/`SchemaFactory.createForClass()` in `book.schema.ts` and `library.schema.ts` (see [03-schemas-and-props.md](03-schemas-and-props.md)).

Two `Book` documents get created and saved first (`this.bookModel.create({...})` twice), each one becoming its own real document in the `books` collection, with MongoDB generating a fresh `_id` for each. Then a `Library` document is created holding those two `_id`s in its `books` array (`books: [book1._id, book2._id]`), and `.save()` writes that third document into the `libraries` collection. This whole sequence is exactly the "referencing" strategy from [07-one-to-one-and-one-to-many-referencing.md](07-one-to-one-and-one-to-many-referencing.md) and [10-relationship-concepts-from-lecture.md](10-relationship-concepts-from-lecture.md): three separate documents, in two separate collections, linked only by ids.

The saved `Library` document (containing the two raw ObjectIds, not the full book data) travels back up through `LibraryService` to `LibraryController`, and Nest's built in response handling serializes whatever a controller method returns into a JSON HTTP response automatically, no manual `res.json(...)` call needed anywhere in this codebase, that is standard NestJS behavior for every controller in every module here.

If you then called `GET /library`, the same request lifecycle would run again, this time reaching `getLibraries()`, which calls `.populate('books')` before returning, meaning the JSON that finally reaches the client would have the full `title`/`author` of each book substituted in place of their bare ObjectIds, the extra query underneath that `.populate()` call is invisible to the client, it just sees a fully assembled result.

## File map

| File | What it teaches | Note |
|---|---|---|
| `src/main.ts` | App bootstrap, `NestFactory.create`, `app.listen` | [01](01-project-setup-and-bootstrap.md) |
| `package.json`, `tsconfig.json`, `nest-cli.json`, `.gitignore` | Project setup, decorator metadata, why no `.env` ships | [01](01-project-setup-and-bootstrap.md) |
| `src/app.module.ts` | `ConfigModule.forRoot`, `MongooseModule.forRoot`, wiring all feature modules | [02](02-connecting-to-mongodb.md) |
| `src/app.controller.ts`, `src/app.service.ts` | The default, untouched Nest CLI starter controller/service | [02](02-connecting-to-mongodb.md) |
| `src/student/student.schema.ts` | `@Schema`, `@Prop`, `SchemaFactory`, the `Student & Document` intersection pattern | [03](03-schemas-and-props.md) |
| `src/user/schemas/user.schema.ts`, `src/employee/schemas/employee.schema.ts`, etc. | The alternate `extends Document` pattern | [03](03-schemas-and-props.md) |
| `src/student/student.module.ts` | `MongooseModule.forFeature` | [04](04-forfeature-and-dependency-injection.md) |
| `src/student/student.service.ts` | `@InjectModel`, `Model<StudentDocument>` | [04](04-forfeature-and-dependency-injection.md) |
| `src/student/student.service.ts`, `src/student/student.controller.ts` | Full CRUD, `find`, `findById`, `findByIdAndUpdate` (merge vs `overwrite: true` replace), `findByIdAndDelete` | [05](05-crud-with-mongoose-models.md) |
| `src/user/schemas/address.schema.ts`, `src/user/schemas/user.schema.ts`, `src/user/user.service.ts` | Embedding a single subdocument | [06](06-embedding-subdocuments.md) |
| `src/product/schemas/tag.schema.ts`, `src/product/schemas/product.schema.ts`, `src/product/product.service.ts` | Embedding an array of subdocuments | [06](06-embedding-subdocuments.md) |
| `src/employee/schemas/profile.schema.ts`, `src/employee/schemas/employee.schema.ts`, `src/employee/employee.service.ts` | One to one referencing, `ref:`, `.populate()` | [07](07-one-to-one-and-one-to-many-referencing.md) |
| `src/library/schemas/book.schema.ts`, `src/library/schemas/library.schema.ts`, `src/library/library.service.ts` | One to many referencing, array of `ObjectId` refs | [07](07-one-to-one-and-one-to-many-referencing.md) |
| `src/project/schemas/project.schema.ts`, `src/project/schemas/developer.schama.ts`, `src/project/project.service.ts` | Many to many referencing, `$set`, `.lean()`, and the `Types.ObjectId` vs `Types.ObjectId[]` type mismatch | [08](08-many-to-many-referencing.md) |
| Every `*.service.spec.ts` and `*.controller.spec.ts` under `src` | Unmodified Nest CLI scaffold tests that fail once real constructor dependencies exist | [09](09-testing-gaps.md) |
| `test/app.e2e-spec.ts` | End to end test, and why it needs a real `MONGO_URI` to pass | [09](09-testing-gaps.md) |
| `lectures/Data Relationships in MongoDB.pdf` | Embedding vs referencing, one to one / one to many / many to many, pros, cons, and when to use each | [10](10-relationship-concepts-from-lecture.md) |

## The single idea underneath all six modules

If you take away one thing from this whole folder, it should be this: every feature module in this repo answers the exact same two questions in a different way. First, does this piece of data belong inside its parent document (embedding) or in its own collection linked by id (referencing)? Second, if referencing, does the reference point one way (`Library` knows about its `Book`s, but not vice versa) or both ways (`Project` and `Developer` each know about the other)? Once you can look at any two related pieces of data in a MongoDB project and answer those two questions for yourself, you have the core modeling skill this entire repository exists to teach, everything else, the `@nestjs/mongoose` decorators, `@InjectModel`, `.populate()`, is just the specific NestJS syntax for putting that answer into working code.
