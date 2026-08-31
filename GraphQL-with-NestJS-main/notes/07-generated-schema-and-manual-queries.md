# 07. The Generated schema.gql, and the queriesForTesting File

This note covers two files that only make sense once you have already read the decorators behind them, `src/schema.gql` and the plain text `queriesForTesting` file sitting at the project root. Together they close the loop between the code first decorators you write and what a real client actually sends over the wire.

## schema.gql is generated, not hand written, confirmed

The very first lines of the file settle this beyond doubt:

```graphql
# ------------------------------------------------------
# THIS FILE WAS AUTOMATICALLY GENERATED (DO NOT MODIFY)
# ------------------------------------------------------
```

This comment is written by `@nestjs/graphql` itself, at the top of every schema file it produces, precisely so nobody mistakes a generated file for hand authored source and starts editing it directly, since any manual edit would simply be overwritten the next time the app starts, because `autoSchemaFile` (covered in [01-project-setup-and-graphql-module.md](01-project-setup-and-graphql-module.md)) tells `GraphQLModule.forRoot()` to rebuild this exact file from the decorators every time.

## The full generated file

```graphql
type Book {
  _id: ID!
  author: String!
  description: String
  title: String!
}

input CreateBookInput {
  author: String!
  description: String
  title: String!
}

type Mutation {
  createBook(input: CreateBookInput!): Book!
  deleteBook(id: String!): Boolean!
  updateBook(input: UpdateBookInput!): Book!
}

type Query {
  getAllBooks: [Book!]!
  getBook(id: String!): Book!
}

input UpdateBookInput {
  author: String
  description: String
  id: ID!
  title: String
}
```

Every one of these five blocks maps directly onto a TypeScript file already covered in earlier notes, and lining them up side by side is the clearest possible proof of how code first GraphQL works in practice. `type Book` comes straight from `book.model.ts`'s `@ObjectType()` class and its four `@Field()` decorated properties (`_id`, `title`, `description`, `author`), covered in [03-the-book-object-type-and-schema.md](03-the-book-object-type-and-schema.md). `input CreateBookInput` and `input UpdateBookInput` come from the two files in `src/book/dto`, covered in [04-input-types-and-validation.md](04-input-types-and-validation.md), including `UpdateBookInput`'s fully optional `title`/`description`/`author` fields generated automatically by `PartialType`. `type Mutation` and `type Query` come from `book.resolver.ts`'s `@Mutation()` and `@Query()` decorated methods, covered in [06-the-resolver-queries-and-mutations.md](06-the-resolver-queries-and-mutations.md), including the exact renaming of `findAll` to `getAllBooks` and `findOne` to `getBook` via each decorator's `{ name: ... }` option. Notice also that the fields inside `type Book` appear alphabetically, `_id`, `author`, `description`, `title`, rather than in the declaration order from `book.model.ts`, which is `sortSchema: true` from `app.module.ts` doing exactly what it is configured to do.

## queriesForTesting: scratch queries meant for the Playground, not for automated tests

Despite living at the project root and having a name that suggests a test suite, `queriesForTesting` is not a Jest file, it has no file extension at all, and reading it confirms it is plain GraphQL query language text, meant to be copied and pasted by a human into an interactive tool, not executed by any test runner in this repository. Its actual content is a handful of GraphQL operations, most of them commented out with a leading `#` (GraphQL's own comment syntax, the same as this project's `.gql` files use), with exactly one left active at the bottom:

```graphql
mutation {
  updateBook(input: {
    id: "687bd02abf3ba6bb62bf2dd6"
    title: "Fullstack development"
    author:"Farzeen Ali"
  }) {
    _id,
    title,
    author
  }
}
```

Above it, commented out, sit four more examples covering every other operation this API exposes: a `createBook` mutation supplying a `title` and `author` inside the `input` argument and asking back for `_id`, `title`, `author`; a `getAllBooks` query asking for the same three fields across every book; a `getBook` query passing a specific hard coded id string; and a `deleteBook` mutation, also with a hard coded id. The hard coded ids scattered through this file, `687bce1ac4d0a8c2ca897315`, `687bcdbac4d0a8c2ca897312`, `687bd02abf3ba6bb62bf2dd6`, are real looking MongoDB ObjectId strings, twenty four character hexadecimal values, exactly the format `_id` takes on once a book has actually been created against a running database, which is itself a clue about how this file was actually used during development: create a book, copy the real `_id` MongoDB assigned it back out of the response, paste that id into the next query down, then comment out the one you just ran and uncomment the next.

That workflow is exactly how a reader would use this file today. With the app running (`npm run start:dev`), opening a browser to `http://localhost:3000/graphql` reaches the interactive GraphQL tool that `playground: true` enables (an in browser IDE for writing and running GraphQL operations against this exact schema, sometimes rendered as the classic GraphQL Playground UI and sometimes as Apollo's own Sandbox UI depending on the exact Apollo Server version in play, either way it is the same idea, a text editor for queries on the left, a run button, and the JSON response on the right). Copying any one block out of `queriesForTesting`, pasting it into that editor, and running it sends exactly that operation to the one `/graphql` endpoint this whole app exposes. Running the `createBook` mutation first is the natural starting point, since every other operation in the file needs a real, existing `_id` to operate on, and the response to that first mutation is where a real id to copy into the next query would come from.
