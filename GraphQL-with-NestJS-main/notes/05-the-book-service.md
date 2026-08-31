# 05. The Book Service: Real MongoDB Storage, Not an In Memory Array

It would be easy to assume a small teaching project like this one keeps its data in a plain in memory array, the kind of `private books: Book[] = []` pattern many introductory examples use. Reading `book.service.ts` in full shows that is not the case here, this service is backed by a real Mongoose model connected to MongoDB, the same storage approach used throughout the MongoDB sibling project in this same folder.

## The whole file

```ts
import { Injectable, NotFoundException } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Book } from './model/book.model';
import { Model } from 'mongoose';
import { CreateBookInput } from './dto/create-book.input';
import { UpdateBookInput } from './dto/update-book.input';

@Injectable()
export class BookService {
    constructor(@InjectModel(Book.name) private bookModel: Model<Book>){}

    async create(input: CreateBookInput): Promise<Book> {
        const created = new this.bookModel(input);
        return created.save();
    }

    async findAll(): Promise<Book[]> {
        return this.bookModel.find().exec();
    }

    async findOne(id: string): Promise<Book> {
        const book = await this.bookModel.findById(id).exec();
        if(!book) throw new NotFoundException('Book not Found!')
        return book;
    }

    async update(input: UpdateBookInput): Promise<Book> {
        const existingBook = await this.bookModel.findById(input.id);
        if(!existingBook) throw new NotFoundException('Book not Found!')
        Object.assign(existingBook, input);
        return existingBook.save();
    }

    async remove(id: string): Promise<boolean> {
        const result = await this.bookModel.findByIdAndDelete(id);
        if(!result) throw new NotFoundException('Book not Found!')
        return true;
    }

}
```

## @InjectModel, the same pattern as the MongoDB sibling project

```ts
constructor(@InjectModel(Book.name) private bookModel: Model<Book>){}
```

This constructor is functionally identical to the pattern the MongoDB sibling project's notes explain in depth for `student.service.ts` and every other feature module there. `@InjectModel(Book.name)` tells Nest's dependency injection container to hand this constructor parameter the specific Mongoose model that was registered under the token `"Book"` (the string value of `Book.name`), which only exists because `book.module.ts` calls `MongooseModule.forFeature([{ name: Book.name, schema: BookSchema }])`. The type, `Model<Book>`, is Mongoose's own generic `Model` type, parameterized with `Book` directly rather than a separate `BookDocument` type, which is possible here specifically because `Book` already `extends Document` (covered in [03-the-book-object-type-and-schema.md](03-the-book-object-type-and-schema.md)), exactly the same reasoning the MongoDB sibling notes give for why `user.service.ts` writes `Model<User>` directly instead of needing an intersection type. Once injected, `this.bookModel` is a real, fully functional Mongoose model, every method called on it below, `.find()`, `.findById()`, `.findByIdAndDelete()`, `new this.bookModel(...)`, is genuine Mongoose API, not anything custom to this project.

## create

```ts
async create(input: CreateBookInput): Promise<Book> {
    const created = new this.bookModel(input);
    return created.save();
}
```

`new this.bookModel(input)` constructs a new, unsaved Mongoose document in memory, using the fields from `CreateBookInput`, `title`, `description`, `author`. Calling `.save()` on it is the actual write, this is the line that sends an insert command to MongoDB and returns a promise resolving to the saved document, complete with its freshly generated `_id`. This is a real database write, not an array push, every book created through this method genuinely persists to whatever MongoDB instance the app's `MONGO_URI` points at.

## findAll

```ts
async findAll(): Promise<Book[]> {
    return this.bookModel.find().exec();
}
```

`this.bookModel.find()` with no filter argument matches every document in the `books` collection, and `.exec()` turns the resulting Mongoose query object into a genuine promise, this is the standard way to run a Mongoose query and get back a real array of documents, resolving to every book currently stored in the database.

## findOne

```ts
async findOne(id: string): Promise<Book> {
    const book = await this.bookModel.findById(id).exec();
    if(!book) throw new NotFoundException('Book not Found!')
    return book;
}
```

`findById(id)` looks a document up by its `_id`. If no document matches (either because the id genuinely does not exist, or because the string passed in is not a validly formatted MongoDB ObjectId, in which case Mongoose can throw a cast error before this line's own `if` check even gets a chance to run), the code checks for a falsy result and throws `NotFoundException`, one of NestJS's built in HTTP exception classes. Worth noting for a beginner: `NotFoundException` is designed with REST in mind, and normally maps to an HTTP 404 status code, but this project uses it from inside a GraphQL resolver's service layer. `@nestjs/graphql` still understands NestJS's built in exception classes when they are thrown from a resolver, and will surface them as a GraphQL error in the response's `errors` array rather than as a raw HTTP status code, so the exception still communicates a meaningful failure to the client, just through GraphQL's own error reporting shape rather than an HTTP status.

## update

```ts
async update(input: UpdateBookInput): Promise<Book> {
    const existingBook = await this.bookModel.findById(input.id);
    if(!existingBook) throw new NotFoundException('Book not Found!')
    Object.assign(existingBook, input);
    return existingBook.save();
}
```

This method first loads the existing document by `input.id`, throwing the same `NotFoundException` if it does not exist, then uses `Object.assign(existingBook, input)` to copy every property from `input`, `id`, and whichever of `title`, `description`, `author` were actually supplied, directly onto the loaded Mongoose document, and finally calls `.save()` to persist those changes. Since `UpdateBookInput` makes every field but `id` optional (via `PartialType`, covered in [04-input-types-and-validation.md](04-input-types-and-validation.md)), a caller who only sends a new `title` will only overwrite `title` on the existing document, `Object.assign` simply never touches keys that were not present on `input` in the first place, so `description` and `author` are left exactly as they were.

## remove

```ts
async remove(id: string): Promise<boolean> {
    const result = await this.bookModel.findByIdAndDelete(id);
    if(!result) throw new NotFoundException('Book not Found!')
    return true;
}
```

`findByIdAndDelete(id)` finds and removes the matching document in one atomic Mongoose call, returning the document that was deleted, or `null` if nothing matched. If nothing matched, the same `NotFoundException` pattern fires. Otherwise the method simply returns `true`, a plain boolean signal that the deletion succeeded, which lines up exactly with `deleteBook`'s `Boolean!` return type in the generated schema, covered in [06-the-resolver-queries-and-mutations.md](06-the-resolver-queries-and-mutations.md) and [07-generated-schema-and-manual-queries.md](07-generated-schema-and-manual-queries.md).

Every one of these five methods is completely unaware that it is being called from a GraphQL resolver rather than a REST controller, `BookService` has no import from `@nestjs/graphql` anywhere in it at all. That separation is itself worth noticing: the service layer's job is talking to the database, full stop, regardless of whether the request that triggered it arrived as a REST call or a GraphQL operation, and `book.resolver.ts` is the only file in this project that actually knows GraphQL is involved on the way in.
