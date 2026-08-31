# Prisma, Neon, and GraphQL with NestJS — Study Guide

This folder holds a complete set of topic by topic notes built from this repository. Where the MongoDB sibling project in this same course was about talking to MongoDB through Mongoose, and the GraphQL sibling project was about building a GraphQL API on top of a plain in memory array, this project fuses the two ideas together and takes the array away entirely. It builds one small GraphQL API for a `Book` resource, and instead of storing books in a JavaScript array that resets every time the server restarts, it stores them in a real PostgreSQL database hosted on Neon, talked to through Prisma, a type safe database toolkit for TypeScript and Node.js.

There is a `lecture` folder with one short slide deck, `Prisma & Neon DB.pdf`, and a `src` folder containing exactly one feature module, `book`, plus a `prisma` folder holding the single file, `schema.prisma`, that declares what the database actually looks like. Every note below explains a concept and then points at the exact file where that concept is used, so you can open the code side by side with the note.

## How to read these notes

Go in order the first time. Each note builds on the one before it.

1. [01-project-setup-and-bootstrap.md](01-project-setup-and-bootstrap.md) — what this project is, `package.json`, `main.ts`, `tsconfig.json`, and the tooling files.
2. [02-what-are-prisma-and-neon-db.md](02-what-are-prisma-and-neon-db.md) — the concepts from the lecture slides, what Prisma is, what Neon is, and why the deck pairs them together.
3. [03-the-prisma-schema-file.md](03-the-prisma-schema-file.md) — `prisma/schema.prisma` in full detail, the generator block, the datasource block, the `Book` model, and a real missing environment variable worth knowing about up front.
4. [04-prismaservice-and-prismamodule.md](04-prismaservice-and-prismamodule.md) — how `PrismaClient` gets wrapped into an injectable NestJS provider, and whether that provider is actually available application wide.
5. [05-book-model-and-input-types.md](05-book-model-and-input-types.md) — `book.model.ts`, `create-book.input.ts`, and `update-book.input.ts`, the GraphQL facing shapes of a book.
6. [06-book-resolver-and-graphql-wiring.md](06-book-resolver-and-graphql-wiring.md) — `book.resolver.ts` and the `GraphQLModule.forRoot` setup in `app.module.ts` that turns those decorated classes into a real, running GraphQL API.
7. [07-book-service-real-database-calls.md](07-book-service-real-database-calls.md) — `book.service.ts`, and exactly how its five methods differ from the equivalent in memory array methods in the GraphQL sibling project.
8. [08-the-generated-schema-gql.md](08-the-generated-schema-gql.md) — `src/schema.gql`, the file Nest writes for you, and what every line in it means.
9. [09-testing-gaps.md](09-testing-gaps.md) — an honest, verified look at every `.spec.ts` file in this repo, including a real bug that goes deeper than a missing test provider.
10. [10-full-recap-and-file-map.md](10-full-recap-and-file-map.md) — one `createBook` mutation traced all the way from an HTTP request down to a real row in Postgres and back, plus a table mapping every file in this repo to the note that explains it.

## A note on the source material, and on verification

The lecture slides in this repo are short, eight slides, TechZeen branded, and mostly bullet points about what Prisma and Neon are and why they pair well together. All of the real depth in these notes comes from reading the actual working code, and, where a claim could be checked rather than assumed, from actually running that code. Several of the bugs and gaps described in these notes, including the exact error text a missing `DATABASE_URL` produces and the exact module resolution failure in three of the four unit test files, were reproduced directly by installing this project's dependencies, generating the Prisma client, and running its test suite and its compiled build, not guessed at from reading alone. Where the code does something a beginner might find surprising, or has a genuine bug or gap, that is called out honestly rather than glossed over.

## A note on the GraphQL sibling project

The sibling project `GraphQL-with-NestJS-main` builds what is functionally the same `Book` API, with the same four fields, the same code first `@ObjectType`/`@InputType`/`@Resolver` decorators, but backed by a plain in memory array living inside its own service class instead of a database. That project's own `notes` folder did not exist yet at the time these notes were written, so this project's GraphQL layer is explained fully and independently here rather than assuming you have already read a comparison. Wherever it is genuinely instructive, these notes still point out exactly how this project's `book.service.ts` differs from what that in memory version almost certainly looks like, since that contrast, an in memory array versus a real Prisma backed Postgres database sitting behind the exact same GraphQL surface, is the entire point of this project existing.
