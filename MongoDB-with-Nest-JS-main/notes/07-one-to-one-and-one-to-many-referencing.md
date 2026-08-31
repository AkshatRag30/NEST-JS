# 07. One to One and One to Many Referencing

The alternative to embedding (the previous note) is referencing: storing related data in its own, completely separate MongoDB collection, and linking the two documents together by storing one document's `_id` inside the other. This repo demonstrates a one to one reference with the `employee`/`profile` module, and a one to many reference with the `library`/`book` module.

## One to one: `Employee` references a `Profile`

```ts
// src/employee/schemas/profile.schema.ts
@Schema()
export class Profile extends Document {
    @Prop()
    age: number;

    @Prop()
    qualification: string;
}
export const ProfileSchema = SchemaFactory.createForClass(Profile);
```

```ts
// src/employee/schemas/employee.schema.ts
@Schema()
export class Employee extends Document {
    @Prop()
    name: string;

    @Prop({ type: MongooseSchema.Types.ObjectId, ref: 'Profile'})
    profile: Profile;
}
export const EmployeeSchema = SchemaFactory.createForClass(Employee);
```

The key difference from the embedding note is right here: `Profile` has its own `SchemaFactory.createForClass(Profile)` call and is registered in `MongooseModule.forFeature([...])` in `employee.module.ts` alongside `Employee`, meaning `Profile` gets its own real MongoDB collection, entirely separate from the `employees` collection.

The `@Prop({ type: MongooseSchema.Types.ObjectId, ref: 'Profile' })` line on `Employee.profile` is what creates the link. `MongooseSchema.Types.ObjectId` (imported here as `Schema as MongooseSchema` from `mongoose`, renamed to avoid clashing with the `@nestjs/mongoose` `Schema` decorator already imported in the same file) tells Mongoose that this field stores a MongoDB ObjectId, the same type every document's `_id` is. `ref: 'Profile'` is the crucial second half: it tells Mongoose that whatever ObjectId ends up in this field is meant to point at a document in the `Profile` collection specifically, which is what makes `.populate('profile')` (below) possible.

```ts
// src/employee/employee.service.ts
async createEmployee(): Promise<Employee>{
    const profile = await new this.profileModel({
        age: 20,
        qualification: 'Masters'
    }).save();
    const employee = new this.employeeModel({
        name: 'Farzeen',
        profile: profile._id
    });
    return employee.save();
}
```

Notice this now takes two separate writes, unlike the single `.save()` calls in the embedding note. First, a standalone `Profile` document is created and saved on its own, and only after that `.save()` resolves does the code know its generated `_id` (`profile._id`). Second, a new `Employee` document is created, storing that `_id` (not the whole profile object) in its `profile` field, and that gets saved as its own separate write. If you looked at the raw `employees` collection in MongoDB directly, the `profile` field there would just contain an ObjectId string, not the age or qualification, those live only in the separate `profiles` collection.

```ts
async findAll(): Promise<Employee[]> {
        return this.employeeModel.find().populate('profile').exec();
    }
```

`.populate('profile')` is the step that turns that raw ObjectId back into a usable object at read time. Without it, `find()` would return employees whose `profile` field is just a bare ObjectId string, useless to display. `.populate('profile')` tells Mongoose "after finding the employees, look at the `ref: 'Profile'` on this field, run a second query against the `profiles` collection for every ObjectId you find, and substitute the real `Profile` document in its place before returning results." This is Mongoose's version of what a SQL database would call a join, except it is implemented as two separate queries under the hood (one for employees, one for profiles) rather than a single database side join, which is exactly the "requires extra queries... slightly slower reads" tradeoff of referencing described in the lecture slides (see [10-relationship-concepts-from-lecture.md](10-relationship-concepts-from-lecture.md)).

## One to many: a `Library` references many `Book`s

```ts
// src/library/schemas/library.schema.ts
@Schema()
export class Library extends Document {
    @Prop()
    name: string;

    @Prop({ type: [{ type: Types.ObjectId, ref: 'Book'}]})
    books: Types.ObjectId[];
}
```

```ts
// src/library/schemas/book.schema.ts
@Schema()
export class Book extends Document {
    @Prop()
    title: string;

    @Prop()
    author: string;
}
```

This is the same referencing idea as `Employee`/`Profile`, extended to a one to many shape: one library can have many books. The type declaration `{ type: [{ type: Types.ObjectId, ref: 'Book' }] }` looks more nested than the employee example, but it is really just the array version of the exact same `{ type: ObjectId, ref: '...' }` pattern, wrapped in an outer array literal (`[...]`), meaning `books` is a list of ObjectIds, each one pointing at a document in the `Book` collection. `Types.ObjectId` here is imported directly from `mongoose`, functionally the same underlying type as `MongooseSchema.Types.ObjectId` used in the employee schema, just accessed through a different (equally valid) import path.

```ts
// src/library/library.service.ts
async createLibrary(): Promise<Library>{
    const book1 = await this.bookModel.create({
        title: 'JS Ka Champion', author: 'Farzeen',
    })
    const book2 = await this.bookModel.create({
        title: 'HTML Ka Champion', author: 'Huzaifa',
    })
    const library = new this.libraryModel({
        name: 'Central Library',
        books: [book1._id, book2._id]
    })
    return library.save();
}
```

`this.bookModel.create({...})` is a Mongoose shorthand equivalent to `new this.bookModel({...}).save()` combined into a single call, it builds and saves a document in one step and resolves with the saved result, which is why `book1` and `book2` already have real `_id`s by the time the code reaches the `books: [book1._id, book2._id]` line. Two independent `Book` documents get created first, and then a `Library` document is created holding an array of their two ObjectIds. This is the referencing equivalent of the embedded array pattern from the `Product`/`Tag` example in the previous note, related items still form a list, but now each item is its own independent, separately queryable document rather than a value nested inside the parent.

```ts
async getLibraries(): Promise<Library[]>{
    return this.libraryModel.find().populate('books');
}
```

`.populate('books')` here works the same way as `.populate('profile')` did above, except since `books` is an array of ObjectIds, Mongoose resolves the entire array into an array of full `Book` documents, matching each ObjectId in the list to its corresponding document in the `books` collection.

## When you would pick this over embedding

Both examples share a trait the embedding examples in the previous note did not: a `Profile` or a `Book` is a meaningful, independently addressable thing on its own, you could plausibly want to query "every book by a certain author" or "this profile" without going through a specific employee or library first. Referencing also avoids one specific problem embedding runs into as data grows: if `Library.books` were embedded instead of referenced, and a library had thousands of books, every single read of that library would have to load every one of those books' full data along with it, even if you only wanted the library's name. Referencing keeps each `Library` document small no matter how many books it points at, at the cost of needing that extra `.populate()` query whenever you actually need the book details, exactly the tradeoff summarized in [10-relationship-concepts-from-lecture.md](10-relationship-concepts-from-lecture.md).
