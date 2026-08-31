# 07. The Book Service and Its Real Database Calls

This is the one file in this project that genuinely differs from what the in memory `GraphQL-with-NestJS-main` sibling project's equivalent service almost certainly looks like. Here is `book.service.ts` in full:

```ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from 'src/prisma/prisma.service';
import { CreateBookInput } from './dto/create-book.input';
import { UpdateBookInput } from './dto/update-book.input';

@Injectable()
export class BookService {
    constructor(private prisma: PrismaService){}

    create(data: CreateBookInput){
        return this.prisma.book.create({ data });
    }

    findAll(){
        return this.prisma.book.findMany();
    }
    findOne(id: string){
        return this.prisma.book.findUnique({
            where: {
                id
            }
        });
    }
    update(data: UpdateBookInput){
        return this.prisma.book.update({
            where: { id: data.id},
            data: {
                title: data.title,
                author: data.author
            }
        })
    }
    remove(id: string){
        return this.prisma.book.delete({ where: {id}})
    }
}
```

`private prisma: PrismaService` is constructor injection receiving the shared, already connected `PrismaService` instance covered in [04-prismaservice-and-prismamodule.md](04-prismaservice-and-prismamodule.md). Everywhere else in this class, `this.prisma.book` is how you reach the generated, model specific set of query methods Prisma built specifically for the `Book` model in `schema.prisma`, `this.prisma.book.create(...)`, `this.prisma.book.findMany(...)`, and so on, each one exactly named after the corresponding real world SQL operation it performs.

## `create`: `this.prisma.book.create({ data })`

An in memory array's version of creating a book would typically look like pushing a new object onto a list held in the service's own memory, something like `this.books.push({ id: randomUUID(), ...data })`, gone forever the moment the process restarts. Here, `this.prisma.book.create({ data })` builds and sends a real SQL `INSERT` statement to the Postgres database, letting Postgres itself generate the `id` (via the `@default(uuid())` attribute from `schema.prisma`) and the `createdAt` timestamp (via `@default(now())`), and the promise this method returns only resolves once that insert has actually completed on the real, remote Neon hosted database, over a real network connection, not a synchronous, instantaneous array push.

## `findAll`: `this.prisma.book.findMany()`

Called with no arguments, `findMany()` matches every row in the `Book` table, exactly a `SELECT * FROM "Book"` executed against Postgres. An in memory equivalent would simply be `return this.books;`, no query, no network round trip, no possibility of the result changing based on what some other process wrote to a shared, durable store, since there would be no shared, durable store at all.

## `findOne`: `this.prisma.book.findUnique({ where: { id } })`

`findUnique` is Prisma's method for looking up exactly one row by a field guaranteed to be unique, here the primary key `id`, functionally the same idea as Mongoose's `findById` from the MongoDB sibling project, translating to a `SELECT * FROM "Book" WHERE id = $1 LIMIT 1` style query. Just like that project's `findById`, this resolves to `null`, not an error, when no row matches, which is exactly the source of the honest gap flagged in the previous note: `book.resolver.ts`'s `getBook` query declares a non nullable `Book!` return type, but this method can genuinely hand it `null`.

## `update`: partial updates without any manual merge logic

```ts
update(data: UpdateBookInput){
    return this.prisma.book.update({
        where: { id: data.id},
        data: {
            title: data.title,
            author: data.author
        }
    })
}
```

This is worth comparing directly against the MongoDB sibling project's `student.service.ts`, which needed two entirely different Mongoose call shapes, a plain merge for `PATCH` and an explicit `overwrite: true` with manual `?? null` fallbacks for `PUT`, to get correct partial versus full replace behavior. Here, there is only one method, and it works correctly for partial updates without any of that extra handling, because of how Prisma's generated `update()` treats its `data` object: any key whose value is `undefined` is simply left untouched on the existing row, only keys with an actual value (including an explicit `null`, for a nullable column) are written. Since `UpdateBookInput`'s `title` and `author` fields come from `PartialType(CreateBookInput)` and are genuinely optional, a caller who only sends `id` and `title` will have `data.author` come through as `undefined` here, and Prisma will correctly leave the existing `author` value on that row alone. There is no equivalent, in this file, of the commented out line and manual fallback dance the MongoDB project's `updateStudent` method needed to get right.

## `remove`: `this.prisma.book.delete({ where: {id}})`

Translating to a real `DELETE FROM "Book" WHERE id = $1`. Worth knowing as a genuine behavioral difference from the Mongoose patterns in the MongoDB sibling project: Mongoose's `findByIdAndDelete` and `findByIdAndUpdate` both quietly resolve to `null` when the target document does not exist. Prisma's `update()` and `delete()` do not behave that way, when the given `where` clause matches no row at all, Prisma throws a runtime exception (its own `P2025`, "record not found", error code) rather than resolving to `null` or `undefined`. Neither `book.service.ts` nor `book.resolver.ts` anywhere catches or translates that exception into a friendlier GraphQL error, so a client calling `updateBook` or `deleteBook` with an `id` that does not exist would see a raw internal server error surface back through the GraphQL response, rather than a clean, deliberate "book not found" message.

## The bigger picture: persistence versus memory

Every one of these five methods, unlike an in memory array's equivalent, is asynchronous for a real reason, not just because the method signature says `Promise`. Each one is a genuine network round trip to a Postgres database that Neon hosts somewhere else on the internet, and every row these methods create, read, change, or remove genuinely persists across this application's own restarts, crashes, and redeployments, because that data lives in a real, durable database rather than in a JavaScript array sitting in this one Node process's memory. That is the entire, concrete difference this project demonstrates relative to the in memory `GraphQL-with-NestJS-main` sibling project, everything above the service layer, the resolver, the model, the input types, the generated schema, is deliberately left almost identical between the two projects specifically so that this one file's difference stands out clearly.
