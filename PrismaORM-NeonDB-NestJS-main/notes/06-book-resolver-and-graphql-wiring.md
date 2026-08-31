# 06. The Book Resolver and How GraphQL Gets Wired Up

A GraphQL resolver plays the same architectural role a controller plays in a REST API: it is the entry point that receives an incoming request (here, a GraphQL query or mutation instead of an HTTP verb and path) and delegates the actual work to a service. This note covers `book.resolver.ts`, and then the `GraphQLModule.forRoot(...)` setup in `app.module.ts` that makes resolvers like this one reachable at all.

## `book.resolver.ts`, the whole file

```ts
import { Args, Mutation, Query, Resolver } from '@nestjs/graphql';
import { Book } from './model/book.model';
import { BookService } from './book.service';
import { CreateBookInput } from './dto/create-book.input';
import { UpdateBookInput } from './dto/update-book.input';


@Resolver(() => Book)
export class BookResolver {
    constructor(private readonly bookService: BookService){}

    @Query(() => [Book])
    getAllBooks() {
        return this.bookService.findAll();
    }

    @Query(() => Book)
    getBook(@Args('id') id: string) {
        return this.bookService.findOne(id);
    }

    @Mutation(() => Book)
    createBook(@Args('input') input: CreateBookInput){
        return this.bookService.create(input);
    }
    @Mutation(() => Book)
    updateBook(@Args('input') input: UpdateBookInput){
        return this.bookService.update(input);
    }
    @Mutation(() => Book)
    deleteBook(@Args('id') id: string){
        return this.bookService.remove(id);
    }
}
```

`@Resolver(() => Book)` marks this class as the resolver responsible for the `Book` GraphQL type, the same way `@Controller('book')` would mark a class as responsible for the `/book` route in a REST API. The constructor takes a `BookService` by constructor injection, exactly the same dependency injection pattern used everywhere else in this course, just requesting a class this project itself wrote instead of a Mongoose model or a raw Prisma client.

`@Query(() => [Book]) getAllBooks()` maps directly onto the `getAllBooks: [Book!]!` entry in the generated `schema.gql`'s `Query` type, and its body is a pure delegation, `return this.bookService.findAll();`, with no logic of its own.

`@Query(() => Book) getBook(@Args('id') id: string)` maps onto `getBook(id: String!): Book!`. `@Args('id')` is how a resolver method pulls one specific named argument out of the incoming GraphQL query's argument list, here binding it to the `id` parameter. This one is worth pausing on: the return type is declared as `Book`, not `Book | null` or a `{ nullable: true }` query option, and the generated schema promises `Book!`, a value that is never null. But `bookService.findOne(id)` (covered in the next note) is backed by Prisma's `findUnique()`, which genuinely resolves to `null` whenever no row matches the given `id`. That is a real, honest mismatch between what this schema promises a client and what the code underneath it can actually produce: if a client queries `getBook` with an `id` that does not exist, the resolver would try to hand GraphQL a `null` value for a field the schema has declared can never be null, which is a genuine error condition for a GraphQL server to be in, rather than a clean, typed "not found" response. Nothing in this resolver or in `book.service.ts` catches that case and turns it into something more deliberate.

`@Mutation(() => Book) createBook(@Args('input') input: CreateBookInput)` maps onto `createBook(input: CreateBookInput!): Book!`, pulling the entire `input` argument object (validated by GraphQL against `CreateBookInput`'s declared shape before this method is ever called) and delegating straight to `bookService.create(input)`. `updateBook` and `deleteBook` follow the identical shape, delegating to `bookService.update(input)` and `bookService.remove(id)` respectively, with `deleteBook` also declared to return a non nullable `Book`, which happens to be safe in practice since Prisma's `delete()` throws an error rather than resolving to `null` when the target row does not exist, covered in the next note.

## How `GraphQLModule.forRoot` makes any of this reachable

None of the decorators above do anything on their own without the GraphQL module being configured in `app.module.ts`:

```ts
GraphQLModule.forRoot<ApolloDriverConfig>({
    driver: ApolloDriver,
    autoSchemaFile: join(process.cwd(), 'src/schema.gql'),
    sortSchema: true,
    playground: true
})
```

`driver: ApolloDriver`, imported from `@nestjs/apollo`, tells Nest's GraphQL module which actual GraphQL server implementation to run underneath, here Apollo Server (the same `apollo-server-express` package listed in `package.json`'s dependencies), the piece of software that actually parses incoming GraphQL query documents, validates them against the schema, and executes the matching resolver.

`autoSchemaFile: join(process.cwd(), 'src/schema.gql')` is what makes this a code first setup rather than a schema first one: instead of a person hand writing a `.graphql` schema definition language file and Nest generating TypeScript classes from it, Nest instead scans every `@ObjectType`, `@InputType`, `@Resolver`, `@Field`, `@Query`, and `@Mutation` decorated class across the whole application at startup, and automatically writes out the equivalent schema as a real file on disk at the given path, `src/schema.gql` in this project (an unusual choice worth noting, many tutorials generate this file at the project root instead; here it is confirmed to live inside `src` by this exact line). `sortSchema: true` tells Nest to alphabetically order the types and fields in that generated file every time it writes it out, which is directly visible in `schema.gql`'s own ordering, `Book`, `CreateBookInput`, `DateTime`, `Mutation`, `Query`, `UpdateBookInput`, exactly alphabetical. `playground: true` turns on Apollo's interactive GraphQL Playground web interface, reachable by visiting `/graphql` in a browser once the app is running, which lets you write and execute real queries and mutations by hand, no separate frontend or API testing tool required.

## The same shape as the in memory sibling, on purpose

Comparing this resolver against what the `GraphQL-with-NestJS-main` sibling project's own `book.resolver.ts` almost certainly looks like, the decorators, the query and mutation names, and the argument shapes are all the same kind of code first GraphQL wiring. That similarity is the entire instructive point of this project existing: a GraphQL API's client facing surface, its queries, its mutations, its input and output shapes, does not need to change at all when the thing sitting behind the resolver changes from a plain in memory array to a real Prisma backed Postgres database. The only file that actually has to change to make that swap is the service the resolver delegates to, covered next.
