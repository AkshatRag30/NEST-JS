# GraphQL with NestJS — Study Guide

This folder holds a complete set of topic by topic notes built from this repository. Unlike the REST focused course project or the MongoDB relationships project sitting next to it on disk, this project is about one specific thing, how NestJS exposes a GraphQL API instead of a REST one, using the `@nestjs/graphql` package together with the Apollo driver from `@nestjs/apollo`. There is a `lectures` folder with one short slide deck, `GraphQL Explained.pdf`, and a single feature module under `src`, the `book` module, which demonstrates every core GraphQL building block NestJS gives you, an object type, input types, a resolver with queries and mutations, and a service that actually talks to MongoDB underneath it all.

Every note below explains a concept and then points at the exact file in this repo where that concept is used, so you can open the code side by side with the note. Every claim in these notes, including which packages are used, whether validation actually runs, and whether the schema file is hand written or generated, was checked directly against this project's real files rather than assumed from how a typical NestJS GraphQL project looks.

## How to read these notes

Go in order the first time. Each note builds on the one before it.

1. [01-project-setup-and-graphql-module.md](01-project-setup-and-graphql-module.md) — what this project is, `package.json`, `main.ts`, and exactly how `GraphQLModule.forRoot()` is configured in `app.module.ts`, including the Apollo driver and the auto generated schema file.
2. [02-graphql-concepts-from-the-slides.md](02-graphql-concepts-from-the-slides.md) — the five slides in `GraphQL Explained.pdf`, explained in plain language, before any code, covering what GraphQL is, why it exists, and how it differs from REST.
3. [03-the-book-object-type-and-schema.md](03-the-book-object-type-and-schema.md) — `book.model.ts`, the `@ObjectType()` and `@Field()` decorators, and how this file does double duty as both a Mongoose schema and a GraphQL type, compared directly to the `@Schema()`/`@Prop()` pattern from the MongoDB sibling project.
4. [04-input-types-and-validation.md](04-input-types-and-validation.md) — `create-book.input.ts` and `update-book.input.ts`, the `@InputType()` decorator as GraphQL's equivalent of a REST DTO, `PartialType`, and a real gap in this project around `class-validator` decorators that are declared but never actually enforced.
5. [05-the-book-service.md](05-the-book-service.md) — `book.service.ts`, the real Mongoose backed storage behind every resolver method, and how it handles create, read, update, and delete.
6. [06-the-resolver-queries-and-mutations.md](06-the-resolver-queries-and-mutations.md) — `book.resolver.ts`, the `@Resolver()`, `@Query()`, `@Mutation()`, and `@Args()` decorators, and how each resolver method becomes one named operation in the schema.
7. [07-generated-schema-and-manual-queries.md](07-generated-schema-and-manual-queries.md) — `schema.gql` itself, proof that it is machine generated rather than hand written, and the `queriesForTesting` file, with an explanation of how you would actually paste those queries into a running GraphQL Playground.
8. [08-testing-gaps.md](08-testing-gaps.md) — an honest look at every `.spec.ts` file in this repo, including which ones would fail the moment you actually run them, and why.
9. [09-full-recap-and-file-map.md](09-full-recap-and-file-map.md) — one complete `createBook` mutation followed by a `getAllBooks` query, traced end to end through every layer, plus a table mapping every file in `src` to the note that explains it.

## A note on the source material

The slide deck in this repo's `lectures` folder is very short, five slides total, and stays at a high conceptual level, what GraphQL is, why it beats REST at avoiding over fetching and under fetching, and the single endpoint idea. It does not get into NestJS specific syntax at all. All of the depth in these notes about decorators, resolvers, and schema generation comes from reading the actual working code in `src`, which is a small but complete, realistic example of a NestJS GraphQL API backed by a real database. Where the code has a genuine gap, such as validation decorators that are never wired up, or a spec file that will fail on the first run, that is called out plainly rather than glossed over.
