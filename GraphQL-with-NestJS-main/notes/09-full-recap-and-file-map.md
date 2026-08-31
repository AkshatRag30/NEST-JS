# 09. Full Recap: One Request, Start to Finish

This last note traces one complete GraphQL exchange through every layer covered in this folder, a `createBook` mutation followed by a `getAllBooks` query, and then gives a single table mapping every file in this project to the note that explains it, so this folder can double as a quick reference after your first read through.

## Tracing a createBook mutation, from the Playground to MongoDB and back

Picture the app running with `npm run start:dev`, and a browser open to `http://localhost:3000/graphql`, running this mutation pasted straight out of the `queriesForTesting` file:

```graphql
mutation {
  createBook(input: {
    title: "Full Stack Dev",
    author: "Farzeen",
  }){
    _id,
    title,
    author
  }
}
```

The browser sends this as a single HTTP POST request to `/graphql`, the one and only endpoint this whole application exposes, exactly the "single endpoint" idea from the slide deck covered in [02-graphql-concepts-from-the-slides.md](02-graphql-concepts-from-the-slides.md). That endpoint exists at all because `AppModule` configured `GraphQLModule.forRoot({ driver: ApolloDriver, ... })` at startup, and the Apollo driver from `@nestjs/apollo` is what actually parses this incoming request body as a GraphQL operation, covered in [01-project-setup-and-graphql-module.md](01-project-setup-and-graphql-module.md).

Apollo, together with `@nestjs/graphql`, checks the operation against the schema built from every decorator in this project, the same schema written out to `src/schema.gql` (covered in [07-generated-schema-and-manual-queries.md](07-generated-schema-and-manual-queries.md)). It finds a `Mutation.createBook` field expecting a `CreateBookInput!` argument, and the `input` object in the request, `{ title: "Full Stack Dev", author: "Farzeen" }`, is checked against that input type's shape, defined in `create-book.input.ts` and covered in [04-input-types-and-validation.md](04-input-types-and-validation.md). GraphQL's own type system confirms `title` and `author` are present and are strings, satisfying `title: String!` and `author: String!`, though as that note explains, the `@IsString()`/`@IsNotEmpty()` `class-validator` decorators on that same class never actually run here, since no `ValidationPipe` is wired up anywhere in this project.

Routing then reaches `BookResolver.createBook` in `book.resolver.ts` (covered in [06-the-resolver-queries-and-mutations.md](06-the-resolver-queries-and-mutations.md)), because `@Mutation(() => Book)` on that method, with no name override, matches the operation name `createBook` exactly. The `input` argument, already parsed into a `CreateBookInput` shaped object by this point, arrives as this method's one parameter, and the resolver does nothing more than delegate: `return this.bookService.create(input);`.

Inside `BookService.create()` (covered in [05-the-book-service.md](05-the-book-service.md)), `this.bookModel`, injected via `@InjectModel(Book.name)` and made possible because `book.module.ts` registered `Book`'s schema with `MongooseModule.forFeature([...])`, constructs a new in memory Mongoose document from `input`, then `.save()` sends the actual insert command to MongoDB, wherever `MONGO_URI` points, and resolves with the saved document, now carrying a real, MongoDB assigned `_id`.

That saved `Book` document travels back up through `BookService`, back through `BookResolver`, and `@nestjs/graphql` serializes it into the exact shape the mutation's selection set asked for, only `_id`, `title`, and `author`, nothing about `description` even though the underlying document technically has that field too (it was simply never supplied, so it is absent), which is the over fetching and under fetching avoidance from [02-graphql-concepts-from-the-slides.md](02-graphql-concepts-from-the-slides.md) working exactly as intended, the client asked for three fields and got back exactly three fields, in one HTTP response, from one endpoint.

## Following it with a getAllBooks query

Running the second query from `queriesForTesting`,

```graphql
query {
  getAllBooks {
    _id,
    title,
    author
  }
}
```

sends another HTTP POST to that same `/graphql` endpoint. This time the schema matches it to `Query.getAllBooks`, which routes to `BookResolver.findAll` because that method's `@Query(() => [Book], { name: 'getAllBooks' })` decorator explicitly renamed it. `findAll` calls `this.bookService.findAll()`, which runs `this.bookModel.find().exec()`, a genuine query across every document in the `books` collection, including the one just created above. The resulting array comes back up through the same two layers and gets serialized the same way, one JSON array, each entry trimmed down to just `_id`, `title`, and `author`, since that is all this particular query's selection set asked for.

## File map

| File | What it teaches | Note |
|---|---|---|
| `src/main.ts` | App bootstrap, `NestFactory.create`, no global `ValidationPipe` | [01](01-project-setup-and-graphql-module.md) |
| `package.json`, `nest-cli.json`, `tsconfig.json`, `.gitignore` | Project setup, the `@nestjs/graphql`/`@nestjs/apollo`/`graphql` package split, why no `.env` ships | [01](01-project-setup-and-graphql-module.md) |
| `src/app.module.ts` | `GraphQLModule.forRoot`, `ApolloDriver`, `autoSchemaFile`, `sortSchema`, `playground`, `MongooseModule.forRoot` | [01](01-project-setup-and-graphql-module.md) |
| `src/app.controller.ts`, `src/app.service.ts`, `src/app.controller.spec.ts` | The untouched, default Nest CLI starter REST controller, still working alongside the GraphQL endpoint | [01](01-project-setup-and-graphql-module.md), [08](08-testing-gaps.md) |
| `lectures/GraphQL Explained.pdf` | What GraphQL is, over fetching and under fetching, the single endpoint idea, queries versus mutations, the schema as a contract | [02](02-graphql-concepts-from-the-slides.md) |
| `src/book/model/book.model.ts` | `@Schema()` plus `@ObjectType()` on one class, `@Field()`, the GraphQL `ID` scalar, comparison to the MongoDB sibling project's `extends Document` pattern | [03](03-the-book-object-type-and-schema.md) |
| `src/book/dto/create-book.input.ts`, `src/book/dto/update-book.input.ts` | `@InputType()` as GraphQL's DTO, `class-validator` decorators present but never enforced, `PartialType` | [04](04-input-types-and-validation.md) |
| `src/book/book.service.ts`, `src/book/book.module.ts` | Real Mongoose backed CRUD, `@InjectModel`, `MongooseModule.forFeature` | [05](05-the-book-service.md) |
| `src/book/resolvers/book.resolver.ts` | `@Resolver()`, `@Query()`, `@Mutation()`, `@Args()`, operation renaming, the `ID` versus `String` inconsistency | [06](06-the-resolver-queries-and-mutations.md) |
| `src/schema.gql`, `queriesForTesting` | The generated schema lined up against every decorator, how to paste example operations into the GraphQL Playground | [07](07-generated-schema-and-manual-queries.md) |
| `src/book/book.service.spec.ts`, `src/book/resolvers/book.resolver.spec.ts`, `test/app.e2e-spec.ts` | Scaffold tests that fail once real constructor dependencies and a real `MONGO_URI` are required | [08](08-testing-gaps.md) |

## The single idea underneath this whole project

If you take one thing away from this folder, it should be this: everything in `@nestjs/graphql` is built around describing your API's shape once, in TypeScript classes with decorators, and letting the framework generate the actual GraphQL schema, the `schema.gql` contract a client relies on, from that single source of truth. `book.model.ts` describes the data. The two files in `book/dto` describe what a client is allowed to send in. `book.resolver.ts` describes what operations exist and what each one does. Nowhere in this project did anyone write GraphQL schema definition language by hand, and nowhere should they, since `autoSchemaFile` regenerates `src/schema.gql` from those decorators every time the app starts. Once you can look at an `@ObjectType()`, an `@InputType()`, and a `@Resolver()` and predict what block of schema each one will produce, you have the core skill this project exists to teach, everything else, the Apollo driver, the Playground, the MongoDB storage underneath, is the supporting cast around that one idea.
