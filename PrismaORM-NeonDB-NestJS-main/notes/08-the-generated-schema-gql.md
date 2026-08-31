# 08. The Generated schema.gql File

`src/schema.gql` is not a file a person in this project ever wrote by hand. As covered in [06-book-resolver-and-graphql-wiring.md](06-book-resolver-and-graphql-wiring.md), it is written automatically by Nest's `GraphQLModule.forRoot({ autoSchemaFile: ... })` setup every time the application starts, generated entirely from the `@ObjectType`, `@InputType`, `@Resolver`, `@Field`, `@Query`, and `@Mutation` decorated classes covered in the previous three notes. Here is the whole file:

```graphql
# ------------------------------------------------------
# THIS FILE WAS AUTOMATICALLY GENERATED (DO NOT MODIFY)
# ------------------------------------------------------

type Book {
  author: String!
  createdAt: DateTime!
  id: String!
  title: String!
}

input CreateBookInput {
  author: String!
  title: String!
}

"""
A date-time string at UTC, such as 2019-12-03T09:54:33Z, compliant with the date-time format.
"""
scalar DateTime

type Mutation {
  createBook(input: CreateBookInput!): Book!
  deleteBook(id: String!): Book!
  updateBook(input: UpdateBookInput!): Book!
}

type Query {
  getAllBooks: [Book!]!
  getBook(id: String!): Book!
}

input UpdateBookInput {
  author: String
  id: String!
  title: String
}
```

## The header comment is a real instruction, not decoration

The very first thing in the file, "THIS FILE WAS AUTOMATICALLY GENERATED (DO NOT MODIFY)", is worth taking literally. Since Nest rewrites this exact file from scratch every time the application starts (because `autoSchemaFile` is configured), any manual edit made directly to `schema.gql` would simply be silently overwritten the next time the app runs. If you ever wanted to change the actual GraphQL API surface, the correct place to do that is back in the decorated TypeScript classes, `book.model.ts`, the two input files, and `book.resolver.ts`, never in this file directly.

## Reading the generated shape against the source classes

`type Book { author: String! createdAt: DateTime! id: String! title: String! }` is the direct translation of `book.model.ts`'s `@ObjectType()` class, with every field marked required (`!`) because plain `@Field()` defaults to non nullable, exactly as explained in [05-book-model-and-input-types.md](05-book-model-and-input-types.md). Notice the fields are listed alphabetically, `author`, `createdAt`, `id`, `title`, rather than in the order they were declared in the TypeScript class (`id`, `title`, `author`, `createdAt`), which is `sortSchema: true` from `GraphQLModule.forRoot(...)` doing exactly what it says.

`input CreateBookInput { author: String! title: String! }` and `input UpdateBookInput { author: String id: String! title: String }` map directly onto the two input files covered in [05-book-model-and-input-types.md](05-book-model-and-input-types.md), and this is the clearest place to see, in one glance, the effect `PartialType(CreateBookInput)` had: `author` and `title` lose their `!` in `UpdateBookInput` (they became optional), while the separately declared `id` field keeps it (it stayed required).

`scalar DateTime`, with its exact description text reproduced from `@nestjs/graphql`'s own built in scalar, is the custom GraphQL scalar automatically registered the moment Nest saw a `Date` typed `@Field()` anywhere in the app (`Book.createdAt`). A GraphQL client receiving a `Book` sees this field serialized as an ISO 8601 style UTC string, exactly matching the example format the description itself gives, `2019-12-03T09:54:33Z`.

`type Mutation { createBook(...) deleteBook(...) updateBook(...) }` and `type Query { getAllBooks getBook(...) }` are the two root types every GraphQL API is required to have at least one of, and they are assembled automatically from every `@Query()` and `@Mutation()` decorated method found across every resolver in the app, here all five coming from the single `BookResolver` class covered in [06-book-resolver-and-graphql-wiring.md](06-book-resolver-and-graphql-wiring.md).

## Where this file lives, and what gets committed to source control

This file is generated inside `src`, at `src/schema.gql`, a slightly unusual placement worth noting since many Nest plus GraphQL tutorials generate this file at the project root instead; it is confirmed here directly from `app.module.ts`'s own `autoSchemaFile: join(process.cwd(), 'src/schema.gql')` line. Unlike the Prisma Client output folder, `/generated/prisma`, which the project's `.gitignore` deliberately excludes from version control (see [03-the-prisma-schema-file.md](03-the-prisma-schema-file.md)), `schema.gql` is not listed in `.gitignore` at all, meaning this particular generated file does get tracked and committed as a real file in this repository. That is a reasonable, deliberate distinction to draw even though the file says not to touch it by hand: `schema.gql` is small, human readable, and genuinely useful to see change in a code review, since it shows exactly how the API's public surface shifted, while the generated Prisma Client in `/generated/prisma` is large, machine generated code that any developer can always regenerate locally with `npx prisma generate` and that adds no value sitting in source control.
