# 04. Input Types: GraphQL's Version of a DTO

REST APIs use DTOs, plain classes decorated with `class-validator` rules, to describe and validate the shape of a request body. GraphQL has its own, very similar concept for the same job, the input type, declared with `@InputType()` instead of a bare class. This project has two of them, `CreateBookInput` and `UpdateBookInput`, both living in `src/book/dto`. This note walks through both files, and then covers a real gap in this project worth understanding clearly: the validation decorators on these classes are declared, but never actually enforced anywhere.

## CreateBookInput

```ts
import { InputType, Field } from "@nestjs/graphql";
import { IsNotEmpty, IsString } from "class-validator";

@InputType()
export class CreateBookInput {
    @Field()
    @IsString()
    @IsNotEmpty()
    title: string;

    @Field({ nullable: true })
    @IsString()
    description?: string;

    @Field()
    @IsString()
    @IsNotEmpty()
    author: string;
}
```

`@InputType()` is GraphQL's equivalent of `@ObjectType()`, but for the other direction, data flowing into the server rather than out of it. A class decorated with `@InputType()` becomes an `input CreateBookInput { ... }` block in the generated schema, and GraphQL's type system treats input types as distinct from object types on purpose, an object type can have fields that resolve through complex logic, but an input type can only ever be plain data, exactly the same restriction a REST DTO has. This is the direct GraphQL analog of, for example, the MongoDB sibling project's REST DTOs, or a `CreateStudentDto` class you would write in a REST controller, same job, different decorator.

Each field pairs a `@Field()` decorator, which controls what shows up in the generated schema, with `class-validator` decorators, `@IsString()` and `@IsNotEmpty()`, which are the exact same validation decorators used throughout the earlier TechZeen REST course project in this same folder structure. `title` and `author` both require a non empty string. `description` is marked `@Field({ nullable: true })`, matching `Book.description`'s own nullability, and only carries `@IsString()`, no `@IsNotEmpty()`, which makes sense, an optional field should not be forced to be non empty when it is provided at all, though notice there is no `@IsOptional()` decorator here either, a detail that becomes relevant below.

## UpdateBookInput and PartialType

```ts
import { CreateBookInput } from "./create-book.input";
import { InputType, Field, PartialType, ID } from "@nestjs/graphql";
import { IsNotEmpty } from "class-validator";

@InputType()
export class UpdateBookInput extends PartialType(CreateBookInput) {
    @Field(() => ID)
    @IsNotEmpty()
    id: string
}
```

`PartialType(CreateBookInput)`, imported here from `@nestjs/graphql` rather than the more commonly seen `@nestjs/mapped-types`, is a utility function that generates a brand new class on the fly, one with every field `CreateBookInput` has, `title`, `description`, `author`, but with every one of them made optional. It does this at both the GraphQL level, marking every field `nullable: true` in the generated schema regardless of how the original field was marked, and at the `class-validator` level, automatically wrapping each inherited validator in an implicit `@IsOptional()`, so a caller can supply just one field to update without the others being rejected as missing. `UpdateBookInput extends PartialType(CreateBookInput)` is the same pattern the earlier TechZeen REST notes describe for REST update DTOs, just imported from the GraphQL flavored package instead of the plain one, since a GraphQL input type needs the `@Field()` metadata regenerated for it too, not just the validator metadata.

On top of that generated partial base, `UpdateBookInput` adds one more required field of its own, `id`, decorated with `@Field(() => ID)` (the same GraphQL `ID` scalar used on `Book._id`, again spelled out explicitly because `id` is typed as a plain `string`) and `@IsNotEmpty()`. This makes sense for an update operation, you need to say which book you are updating, and every other field, `title`, `description`, `author`, is optional, since a client updating just the title should not be forced to resend the author too. Looking at the generated schema confirms this shape exactly:

```graphql
input UpdateBookInput {
  author: String
  description: String
  id: ID!
  title: String
}
```

`id: ID!` is the only non nullable field, everything else has no `!`, meaning optional, exactly matching `PartialType`'s behavior plus the one manually added required `id` field.

## The gap: these validators are never actually enforced

Here is the part worth being direct about. Both input classes are fully decorated with real `class-validator` rules, and it would be easy to assume, reading only these two files, that sending a `createBook` mutation with an empty `title` would be rejected before it ever reached `BookService`. That is not what happens in this project, and the reason is visible by checking `src/main.ts` in full:

```ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT ?? 3000);
}
```

There is no `app.useGlobalPipes(new ValidationPipe())` call anywhere in this file, and no `@UsePipes(ValidationPipe)` decorator on `BookResolver` or any of its individual methods either (confirmed by reading `book.resolver.ts` in full in [06-the-resolver-queries-and-mutations.md](06-the-resolver-queries-and-mutations.md)). In NestJS, `class-validator` decorators on a DTO or input class do nothing by themselves, they are inert metadata until something, almost always Nest's built in `ValidationPipe`, actually reads that metadata and runs the checks against an incoming payload. REST controllers in NestJS apps typically get this wired up either globally in `main.ts` or per route with `@UsePipes()`. GraphQL resolvers need the exact same wiring, `@nestjs/graphql` does not run `class-validator` automatically just because an `@InputType()` class happens to also carry `@IsString()`/`@IsNotEmpty()` decorators.

Since neither wiring exists anywhere in this project, sending a `createBook` mutation with `title: ""` or with `title` omitted entirely (if the schema allowed it, which it does not since `title: String!` is non nullable at the GraphQL type level) would sail straight through to `BookService.create()` with no `class-validator` check ever running. The only validation actually happening at all here is GraphQL's own type level checking, which enforces that non nullable fields (`title: String!`, `author: String!`) must be present and must be strings, but says nothing at all about an empty string being invalid, since an empty string is still a perfectly valid `String` as far as the GraphQL type system is concerned. So `@IsNotEmpty()` on `title`, specifically, currently has zero actual effect on this application's behavior. This is a genuine, checkable gap in the project, not a guess, and the fix would be a single line added to `main.ts`, `app.useGlobalPipes(new ValidationPipe())`, the same fix a REST NestJS project would need for the same reason.
