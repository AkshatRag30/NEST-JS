# 01. Project Setup and How GraphQLModule Is Configured

This is a standard Nest CLI generated project, the same `main.ts`, `nest-cli.json`, and `tsconfig.json` shape you would see in any fresh `nest new` app, with one feature module, `book`, and a GraphQL layer wired into the root `AppModule`. This note walks through the setup files first, then goes deep on the one line of configuration that makes this whole project a GraphQL API rather than a REST one.

## package.json, and which packages actually matter here

Looking at the dependencies in `package.json`, three packages are doing the GraphQL specific work:

```json
"@nestjs/apollo": "^13.1.0",
"@nestjs/graphql": "^13.1.0",
"apollo-server-express": "^3.13.0",
"graphql": "^16.11.0",
```

`graphql` is the underlying, framework agnostic JavaScript implementation of the GraphQL specification itself, it is not a NestJS package at all, it is the same library any GraphQL server in the Node ecosystem would depend on. `@nestjs/graphql` is NestJS's own integration layer, it is what gives you decorators like `@ObjectType()`, `@Field()`, `@Resolver()`, `@Query()`, and `@Mutation()`, letting you describe a GraphQL schema using TypeScript classes instead of writing schema files by hand. `@nestjs/apollo` is the specific driver that plugs the popular Apollo Server implementation into `@nestjs/graphql`, since NestJS's GraphQL module is deliberately driver agnostic, it can run on top of Apollo or on top of an alternative like Mercurius, and this project has chosen Apollo. `apollo-server-express` is Apollo Server's own package for running inside an Express based HTTP server, which lines up with `@nestjs/platform-express` also being a dependency here, this app runs on Express under the hood, the same as any default NestJS app, with Apollo layered on top to handle the single `/graphql` endpoint.

Beyond the GraphQL stack, `@nestjs/mongoose` and `mongoose` are present too, which tells you up front that this project's data does not live in memory, it is backed by a real MongoDB database, a detail worth keeping in mind before you even open `book.service.ts` (covered in full in [05-the-book-service.md](05-the-book-service.md)). `class-validator` and `class-transformer` are also dependencies, which matters for [04-input-types-and-validation.md](04-input-types-and-validation.md), where it turns out they are only half wired up.

## main.ts, a completely default bootstrap

```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

There is nothing GraphQL specific here at all, and that is exactly the point of how `@nestjs/graphql` is designed, all of the GraphQL wiring happens declaratively inside `AppModule`'s `imports` array, not here in the bootstrap function. `NestFactory.create(AppModule)` builds the whole application, including its GraphQL endpoint, and `app.listen()` starts the one Express server that serves both any regular HTTP routes and the `/graphql` endpoint together. Notice also that there is no `app.useGlobalPipes(new ValidationPipe())` call anywhere in this file, which becomes important later, hold onto that fact for [04-input-types-and-validation.md](04-input-types-and-validation.md).

## AppModule, where the real configuration lives

```ts
@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
      autoSchemaFile: join(process.cwd(), 'src/schema.gql'),
      sortSchema: true,
      playground: true
    }),
    MongooseModule.forRoot(process.env.MONGO_URI!),
    BookModule,
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

`ConfigModule.forRoot({ isGlobal: true })` comes from `@nestjs/config`, and marking it global means every other module in the app, including `BookModule`, can read environment variables through `ConfigService` (or, as this project actually does it, through `process.env` directly) without importing `ConfigModule` again itself.

`GraphQLModule.forRoot<ApolloDriverConfig>({...})` is the one line that turns this into a GraphQL server. `driver: ApolloDriver` tells `@nestjs/graphql` to use the Apollo driver from `@nestjs/apollo` rather than any other GraphQL server implementation, this is the piece that actually creates the `/graphql` endpoint and handles incoming GraphQL requests over HTTP. The generic type parameter `ApolloDriverConfig` is purely a TypeScript convenience, it makes the options object that follows type checked against exactly the options Apollo's driver understands, so a typo in an option name would be caught at compile time.

`autoSchemaFile: join(process.cwd(), 'src/schema.gql')` is the setting that answers the exact question this project raises just by having a `schema.gql` file sitting in `src`: is that file hand written, or generated. This option tells `@nestjs/graphql` to run in what its documentation calls "code first" mode, meaning you never write GraphQL schema definition language yourself, instead you write TypeScript classes decorated with `@ObjectType()`, `@InputType()`, `@Field()`, and so on (exactly what `book.model.ts` and the two input files do), and at application startup `@nestjs/graphql` inspects all of that decorator metadata, builds the equivalent schema in memory, and writes it out as a real `.gql` file at the path given here, `src/schema.gql`. That means `schema.gql` in this repo is not something a developer typed by hand, it is a build artifact that gets regenerated every time the app starts. The comment at the very top of the real file confirms this directly:

```graphql
# ------------------------------------------------------
# THIS FILE WAS AUTOMATICALLY GENERATED (DO NOT MODIFY)
# ------------------------------------------------------
```

[03-the-book-object-type-and-schema.md](03-the-book-object-type-and-schema.md) and [07-generated-schema-and-manual-queries.md](07-generated-schema-and-manual-queries.md) both come back to this and line the generated file up field by field against the decorators that produced it, so you can see the code first workflow with your own eyes rather than just taking the comment's word for it.

`sortSchema: true` simply tells the schema generator to alphabetize types and fields in the output file, which is why, if you look at the real `schema.gql`, `Book`'s fields appear as `_id`, `author`, `description`, `title`, alphabetical order rather than the order they were declared in `book.model.ts`. It is a cosmetic setting, purely about making the generated file's diffs stable and easy to read, it has no effect on how the API behaves.

`playground: true` enables Apollo's interactive, in browser GraphQL exploration tool, reachable in development by opening the `/graphql` endpoint directly in a browser rather than sending it a POST request. This is what [07-generated-schema-and-manual-queries.md](07-generated-schema-and-manual-queries.md) assumes you will use to actually try the queries and mutations written in the `queriesForTesting` file.

`MongooseModule.forRoot(process.env.MONGO_URI!)` connects the whole application to MongoDB at startup, using a connection string read from the `MONGO_URI` environment variable. The trailing `!` is a TypeScript non null assertion, it tells the compiler "trust me, this will not be undefined," but it does nothing at runtime to guarantee that. This repository ships no `.env` file (its `.gitignore` explicitly excludes `.env` and every variant of it), so `MONGO_URI` will only actually have a value if you create your own local `.env` file or otherwise set that environment variable yourself before starting the app. Without it, `MongooseModule.forRoot(undefined)` will fail to establish a real database connection, which matters for [08-testing-gaps.md](08-testing-gaps.md), since the end to end test bootstraps this exact module.

Finally, `BookModule` is imported the same way any feature module is imported into `AppModule`, nothing GraphQL specific about that line itself, the GraphQL specific work inside `BookModule` is covered in [05-the-book-service.md](05-the-book-service.md) and [06-the-resolver-queries-and-mutations.md](06-the-resolver-queries-and-mutations.md).

## The leftover REST controller

`AppController` and `AppService` are the completely untouched, default files the Nest CLI scaffolds into every new project:

```ts
@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get()
  getHello(): string {
    return this.appService.getHello();
  }
}
```

This is a plain REST style route, `GET /`, returning the string `Hello World!`. It has nothing to do with GraphQL, it exists purely because it was never deleted after the project was generated, and it is a useful reminder that a NestJS app can serve REST routes and a GraphQL endpoint side by side in the same running server, they are not mutually exclusive, `@nestjs/graphql` simply adds one more route, `/graphql`, on top of whatever REST controllers already exist.
