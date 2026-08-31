# 03. The Prisma Schema File, in Full Detail

Everything Prisma knows about your database in this project comes from a single file, `prisma/schema.prisma`. It is short enough to read in full:

```prisma
// This is your Prisma schema file,
// learn more about it in the docs: https://pris.ly/d/prisma-schema

// Looking for ways to speed up your queries, or scale easily with your serverless or edge functions?
// Try Prisma Accelerate: https://pris.ly/cli/accelerate-init

generator client {
  provider = "prisma-client-js"
  output   = "../generated/prisma"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Book {
  id String @id @default(uuid())
  title String
  author String
  createdAt DateTime @default(now())
}
```

The first two comment blocks are just the default Prisma CLI scaffold text, pointing at Prisma's own documentation and at Prisma Accelerate, a separate paid product for connection pooling and caching that this project does not actually use anywhere. Below those comments, the file has exactly three real parts.

## The generator block

```prisma
generator client {
  provider = "prisma-client-js"
  output   = "../generated/prisma"
}
```

A generator block tells the Prisma CLI what kind of code to produce when you run `npx prisma generate`. `provider = "prisma-client-js"` means generate the actual Prisma Client, the fully typed TypeScript/JavaScript library your application code imports and calls methods on, as opposed to Prisma's other generators (for example ones that produce a GraphQL SDL file or Zod validation schemas directly from your models, neither of which this project uses).

`output = "../generated/prisma"` is the detail worth slowing down on. By default, a freshly generated Prisma Client would be placed inside `node_modules`, which is why most Prisma tutorials simply write `import { PrismaClient } from '@prisma/client'`. This project overrides that default, telling Prisma to instead write the generated client's code into a folder at `../generated/prisma` relative to this schema file's own location (`prisma/schema.prisma`), which resolves to a top level `generated/prisma` folder sitting right next to `src` at the project root. This is exactly why the `.gitignore` note in the previous chapter listed `/generated/prisma`, that folder does not exist until someone runs `npx prisma generate` locally, and it is exactly why `prisma.service.ts` (covered in the next note) imports `PrismaClient` from the literal path `'generated/prisma'` rather than from the `@prisma/client` package name.

## The datasource block

```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}
```

A datasource block tells Prisma which actual database engine to talk to and how to connect to it. `provider = "postgresql"` confirms, at the schema level, exactly what the lecture slides described: this project targets a real PostgreSQL database, the same kind of database Neon hosts, not MongoDB, MySQL, or SQLite.

`url = env("DATABASE_URL")` is the connection string Prisma will actually use to open a connection, and `env(...)` is Prisma's own schema syntax for reading that value out of an environment variable rather than hardcoding a real connection string (with a real username and password in it) directly into a file that gets committed to source control.

This is exactly the same category of gap flagged in the MongoDB sibling project's missing `MONGO_URI`, and it was checked here the same way, by actually searching this entire repository for the string `DATABASE_URL`. It appears in exactly one place: this line in `schema.prisma`. There is no `.env` file, no `.env.example` file, and no other reference to `DATABASE_URL` anywhere in this codebase, not in `README.md`, not in any script, nowhere. Whoever runs this project is expected to create their own local `.env` file containing a real Neon connection string before this app can talk to a real database.

What actually happens without one is worth being precise about, since it was verified directly rather than assumed: constructing `new PrismaClient()` does not throw immediately, even with no `DATABASE_URL` set at all. The failure only happens the moment something actually tries to use the connection, for example calling `$connect()`. Running that call with `DATABASE_URL` deliberately unset in this exact project produces this real error, reproduced word for word:

```
error: Environment variable not found: DATABASE_URL.
  -->  schema.prisma:14
   |
13 |   provider = "postgresql"
14 |   url      = env("DATABASE_URL")
   |

Validation Error Count: 1
```

That distinction, that the missing variable is only discovered lazily at connection time rather than the moment the class is instantiated, matters directly for understanding why one particular test in this project's suite behaves the way it does, covered in [09-testing-gaps.md](09-testing-gaps.md).

## The model block: `Book`

```prisma
model Book {
  id String @id @default(uuid())
  title String
  author String
  createdAt DateTime @default(now())
}
```

A `model` block in Prisma is the declarative description of one database table, and conceptually it maps almost one to one onto a real SQL table: the model's name, `Book`, becomes the table's name, and each field listed inside becomes one column in that table, with its own SQL type and constraints, generated automatically from what you write here.

`id String @id @default(uuid())` declares a field named `id`, typed as `String`. The `@id` attribute marks this field as the table's primary key, the unique value Postgres and Prisma both use internally to identify one specific row (this is Prisma's equivalent of MongoDB's automatic `_id`, except here you are declaring it explicitly rather than getting it for free). `@default(uuid())` tells Prisma to generate a random universally unique identifier, a long unique string like `550e8400-e29b-41d4-a716-446655440000`, for this field automatically whenever a new `Book` row is created and no `id` was supplied, which is exactly why neither `create-book.input.ts` nor `book.resolver.ts`'s `createBook` mutation ever asks a caller to provide one (see [05-book-model-and-input-types.md](05-book-model-and-input-types.md)).

`title String` and `author String` are both plain required text columns, with no default value and no attribute beyond their type, meaning Postgres will reject an attempt to insert a `Book` row missing either one.

`createdAt DateTime @default(now())` is a timestamp column that Prisma automatically fills in with the current date and time, via the built in `now()` function, at the moment a row is inserted, with no need for the calling code to ever supply a value for it. This is a narrower version of what Mongoose's `{ timestamps: true }` schema option did automatically for the MongoDB sibling project's models, except here it only covers creation time. There is no matching `updatedAt` field on this model at all, and note that this repository never uses Prisma's own dedicated `@updatedAt` attribute (a real Prisma feature that would auto refresh a timestamp column every time a row is updated) anywhere either. That means if a `Book` row is later changed through `book.service.ts`'s `update()` method (covered in [07-book-service-real-database-calls.md](07-book-service-real-database-calls.md)), nothing in this schema records when that change happened.

It is also worth being explicit, since the task of documenting this project specifically asked to check for it, that no field on this model carries a `@unique` attribute anywhere. `id` is unique by virtue of being the primary key, but neither `title` nor `author` has any uniqueness constraint, so nothing in the database itself would stop two different rows from having the exact same title and author.

## What a saved `Book` looks like as a real table

Putting all three parts together, once this schema has been pushed or migrated to a real Postgres database, it produces one real table, conceptually shaped like this:

| id (text, primary key) | title (text, not null) | author (text, not null) | createdAt (timestamp, defaults to now) |
|---|---|---|---|

Every `Book` object your application code works with through Prisma's generated client, covered in the next note, is really just one row read from or written to that one table over a real network connection to Neon's hosted Postgres server, which is a fundamentally different kind of storage than the plain in memory array the sibling `GraphQL-with-NestJS-main` project uses for the exact same shaped `Book` resource, a contrast made concrete with real database calls in [07-book-service-real-database-calls.md](07-book-service-real-database-calls.md).
