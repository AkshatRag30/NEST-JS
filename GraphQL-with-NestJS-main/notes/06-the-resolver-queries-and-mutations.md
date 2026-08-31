# 06. The Resolver: Queries, Mutations, and Args

If `book.model.ts` is GraphQL's version of a schema, `book.resolver.ts` is GraphQL's version of a controller. This is the file that actually receives an incoming GraphQL operation, decides which service method should handle it, and hands back the result. This note goes through every method in it and lines each one up against the corresponding line in the generated `schema.gql`.

## The whole file

```ts
import { Args, Mutation, Query, Resolver } from '@nestjs/graphql';
import { BookService } from '../book.service';
import { Book } from '../model/book.model';
import { CreateBookInput } from '../dto/create-book.input';
import { UpdateBookInput } from '../dto/update-book.input';

@Resolver(() => Book)
export class BookResolver {
    constructor(private readonly bookService: BookService){}

    @Query(() => [Book], { name: 'getAllBooks' })
    async findAll(){
        return this.bookService.findAll();
    }

    @Query(() => Book, { name: 'getBook' })
    async findOne(@Args('id', { type: () => String }) id:string ){
        return this.bookService.findOne(id);
    }

    @Mutation(() => Book)
    async createBook(@Args('input') input: CreateBookInput) {
        return this.bookService.create(input);
    }

    @Mutation(() => Book)
    async updateBook(@Args('input') input: UpdateBookInput) {
        return this.bookService.update(input);
    }

    @Mutation(() => Boolean)
    async deleteBook(@Args('id', { type: () => String }) id: string ) {
        return this.bookService.remove(id);
    }
}
```

## @Resolver, the class level decorator

```ts
@Resolver(() => Book)
export class BookResolver {
```

`@Resolver()` is what marks a class as a GraphQL resolver at all, the equivalent role `@Controller()` plays for REST routing. The argument, `() => Book`, tells `@nestjs/graphql` which object type this resolver is primarily concerned with, useful context for more advanced GraphQL features like field level resolvers, which this project does not use (every field on `Book` here is a plain data field with no custom resolution logic of its own, so this argument is mostly declarative bookkeeping in this particular file). The constructor injects `BookService` the standard NestJS way, constructor based dependency injection, no different from how a controller would inject a service.

## Queries: getAllBooks and getBook

```ts
@Query(() => [Book], { name: 'getAllBooks' })
async findAll(){
    return this.bookService.findAll();
}
```

`@Query()` marks this method as a GraphQL query, a read only operation. The first argument, `() => [Book]`, tells the schema generator this operation returns an array of `Book` objects. The second argument, `{ name: 'getAllBooks' }`, is what actually names the operation in the schema, overriding what would otherwise default to the method's own name, `findAll`. This decoupling between the TypeScript method name and the public GraphQL operation name is exactly why the generated schema shows `getAllBooks` as the operation name even though nowhere in this file is a method literally called `getAllBooks`. Checking `schema.gql` confirms this line by line:

```graphql
type Query {
  getAllBooks: [Book!]!
  getBook(id: String!): Book!
}
```

`getAllBooks: [Book!]!` reads as "an array of `Book`, where the array itself is never null, and no individual element inside it is ever null either," which is the schema generator's default non nullable behavior applied to the `[Book]` return type given in the decorator.

```ts
@Query(() => Book, { name: 'getBook' })
async findOne(@Args('id', { type: () => String }) id:string ){
    return this.bookService.findOne(id);
}
```

`getBook`, similarly renamed from the method `findOne`, returns a single `Book`. `@Args('id', { type: () => String })` is how a GraphQL operation receives an argument, the string `'id'` is the argument's name as it will appear in a query (matching the `id: String!` shown in `schema.gql` above), and `{ type: () => String }` tells the schema generator this argument's GraphQL type is `String`, again wrapped in a function for the same reason seen in `book.model.ts`, a plain primitive type annotation on its own is not always enough information for the schema builder to infer the exact GraphQL scalar intended. Whatever value the client sends for `id` gets passed straight through as the `id` parameter into `findOne`, which then simply forwards it to `bookService.findOne(id)`.

## Mutations: createBook, updateBook, deleteBook

```ts
@Mutation(() => Book)
async createBook(@Args('input') input: CreateBookInput) {
    return this.bookService.create(input);
}
```

`@Mutation()` marks an operation as one that changes data, matching exactly the query versus mutation distinction covered conceptually in [02-graphql-concepts-from-the-slides.md](02-graphql-concepts-from-the-slides.md). Here, no explicit `name` option is given, so the mutation's public name in the schema falls back to the method's own name, `createBook`, which happens to already be the name wanted, unlike the two queries above. `@Args('input')` here has no explicit `type` option, because `input: CreateBookInput` gives the schema generator everything it needs already, `CreateBookInput` is itself a class decorated with `@InputType()`, so its shape is already fully described elsewhere, there is no ambiguity to resolve the way there was with a bare `string` typed `id` argument. The generated schema reflects this exactly:

```graphql
type Mutation {
  createBook(input: CreateBookInput!): Book!
  deleteBook(id: String!): Boolean!
  updateBook(input: UpdateBookInput!): Book!
}
```

`updateBook` follows the identical shape, taking an `UpdateBookInput` and returning the updated `Book`. `deleteBook` takes a plain `id: String` argument (using the same explicit `{ type: () => String }` pattern as `getBook`) and returns a `Boolean`, matching `BookService.remove()`'s own `Promise<boolean>` return type exactly.

## A detail worth noticing: ID versus String for the same identifier

Look closely at how a book's identifier is typed across this project, and a small inconsistency shows up. `Book._id` is declared `@Field(() => ID)`, using GraphQL's dedicated `ID` scalar. `UpdateBookInput.id` is also declared `@Field(() => ID)`. But `getBook`'s and `deleteBook`'s own `id` arguments are both declared `{ type: () => String }`, the plain `String` scalar, not `ID`. The generated schema preserves this exact split, `getBook(id: String!)` and `deleteBook(id: String!)` both use `String`, while `Book`'s own `_id: ID!` and `UpdateBookInput`'s `id: ID!` both use `ID`. Functionally this causes no actual failures, `ID` and `String` are both serialized identically as quoted text over the wire, and a client can send the exact same value either way without noticing a difference, but it is an inconsistency in this codebase's own typing choices worth being aware of, the same identifier concept is represented by two different GraphQL scalars depending on which operation you look at, rather than one scalar used consistently everywhere an id appears.

Every resolver method here follows the same shallow shape, receive already validated by GraphQL's own type system input, and immediately hand off to the matching `BookService` method with no extra logic of its own, keeping the resolver a thin routing layer, exactly the same separation of concerns a REST controller is meant to have from its service.
