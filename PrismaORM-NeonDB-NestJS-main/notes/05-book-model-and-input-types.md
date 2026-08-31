# 05. The Book GraphQL Model and Its Input Types

This project uses NestJS's code first approach to GraphQL, meaning you write plain TypeScript classes decorated with special decorators from `@nestjs/graphql`, and Nest generates the actual GraphQL schema definition language file from those classes automatically (that generated file, `src/schema.gql`, gets its own full explanation in [08-the-generated-schema-gql.md](08-the-generated-schema-gql.md)). This is the exact same approach the sibling `GraphQL-with-NestJS-main` project uses for its own in memory `Book` type, which is why the three files in this note will likely look almost identical to that project's equivalents, the whole point of this project is that the GraphQL facing shape of a book barely has to change at all when the storage underneath changes from an array to a real database.

## The object type: `book.model.ts`

```ts
import { ObjectType, Field } from "@nestjs/graphql";

@ObjectType()
export class Book {
    @Field()
    id: string;

    @Field()
    title: string;

    @Field()
    author: string;

    @Field()
    createdAt: Date;
}
```

`@ObjectType()` marks this class as a GraphQL object type, meaning it describes a shape of data that can be returned as the result of a query or mutation, exactly matching the `type Book { ... }` block you will find in the generated `schema.gql`. `@Field()` on each property is what actually exposes that property as a field on the GraphQL type. Nest infers the underlying GraphQL scalar type from the TypeScript type where it reasonably can, `string` maps to GraphQL's built in `String` scalar, and `Date` maps to a custom `DateTime` scalar that `@nestjs/graphql` registers automatically the first time it sees a `Date` typed field anywhere in the app (this same `DateTime` scalar shows up as its own entry in `schema.gql`, explained fully in that note).

None of these four fields is marked as nullable, and by default `@nestjs/graphql` treats every plain `@Field()` as required, non nullable, in the generated schema, which is exactly reflected in `schema.gql`'s `type Book { author: String! createdAt: DateTime! id: String! title: String! }`, every field carrying the `!` that means "this will never be null." Whether that promise is actually kept everywhere a `Book` gets returned is a real question worth asking, and it comes up directly in the next note when looking at `book.resolver.ts`'s `getBook` query.

## The create input: `create-book.input.ts`

```ts
import { InputType, Field } from "@nestjs/graphql";
@InputType()
export class CreateBookInput {
    @Field()
    title: string;

    @Field()
    author: string
}
```

`@InputType()` is the counterpart to `@ObjectType()` for the other direction of data flow: an input type describes the shape of data a GraphQL client is allowed to send in as an argument to a mutation, and, unlike an object type, an input type can never itself be returned as a query or mutation's result.

Notice `CreateBookInput` only has `title` and `author`, nothing else. There is no `id` field here, and no `createdAt` field either, and that omission is entirely intentional given what you already know from the `schema.prisma` model in the previous note: `id` gets generated automatically by the database via `@default(uuid())`, and `createdAt` gets stamped automatically via `@default(now())`. A client creating a new book has nothing useful to say about either of those two columns, so this input type simply never asks for them, which is exactly why `schema.gql`'s `input CreateBookInput` only lists `author: String!` and `title: String!`.

## The update input: `update-book.input.ts`

```ts
import { InputType, Field, PartialType } from "@nestjs/graphql";
import { CreateBookInput } from "./create-book.input";
@InputType()
export class UpdateBookInput extends PartialType(CreateBookInput) {
    @Field()
    id: string
}
```

`PartialType(CreateBookInput)` is a NestJS GraphQL utility function, and it is worth being precise about what it actually does: it generates, at runtime, a brand new class containing every field `CreateBookInput` has, `title` and `author`, but with every one of them made optional instead of required. `UpdateBookInput` then `extends` that generated class and adds one more field of its own, `id`, declared directly with a plain `@Field()`, which stays required since it was never passed through `PartialType`'s optional wrapping.

The result is exactly what `schema.gql`'s `input UpdateBookInput` shows: `author: String`, `id: String!`, `title: String`, only `id` carries the `!` that marks it required, `author` and `title` are both genuinely optional, letting a caller send just the one field they actually want to change alongside the `id` of the book they mean to change. This design decision, letting title and author be independently optional on update, lines up directly with how `book.service.ts`'s `update()` method actually calls Prisma, covered in [07-book-service-real-database-calls.md](07-book-service-real-database-calls.md), where Prisma's own update semantics turn out to already handle a partially filled in object correctly without any extra work, a genuinely simpler situation than the manual `PUT` versus `PATCH` handling the MongoDB sibling project's Mongoose based `student` module needed.
