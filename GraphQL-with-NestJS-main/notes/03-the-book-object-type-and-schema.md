# 03. The Book Object Type: book.model.ts

`src/book/model/book.model.ts` is the single most important file in this project to understand well, because it does two jobs at once inside one class, and seeing both jobs clearly is the key to understanding everything that follows it.

## The whole file

```ts
import { Prop, Schema, SchemaFactory } from "@nestjs/mongoose";
import { Document } from "mongoose";
import { ObjectType, Field, ID } from "@nestjs/graphql";

@Schema()
@ObjectType()
export class Book extends Document {
    @Field(() => ID)
   declare readonly _id: string;

   @Prop({ required: true })
   @Field()
   title: string;

   @Prop()
   @Field({ nullable: true })
   description?: string;

   @Prop({ required: true })
   @Field()
   author: string
}

export const BookSchema = SchemaFactory.createForClass(Book);
```

## Two decorators stacked on one class

The class `Book` carries both `@Schema()` from `@nestjs/mongoose` and `@ObjectType()` from `@nestjs/graphql` at the same time. These two decorators come from entirely different libraries and serve entirely different purposes, but NestJS lets you stack them on the same class because decorators just attach metadata, and nothing stops two unrelated packages from each attaching their own metadata to the same class. `@Schema()` marks `Book` as a Mongoose schema definition, the same role it plays in the MongoDB sibling project's `student.schema.ts` and every other schema file there. `@ObjectType()` marks `Book` as a GraphQL object type, meaning it becomes a `type Book { ... }` block in the generated GraphQL schema, the actual shape of data a client can ask for. One class, two responsibilities: it is both the thing that gets saved to and read from MongoDB, and the thing a GraphQL client sees and queries against.

This is a real, deliberate choice in this project, and it is worth comparing directly to how the MongoDB sibling project handles the same kind of file. Over there, `student.schema.ts` defines `Student` purely as a Mongoose schema, with a separate `StudentDocument` type built via `Student & Document` intersection, and REST DTOs for creating or updating a student would, in a REST project, typically live in their own separate classes. Here, the same `Book` class is reused directly as the GraphQL response type, there is no separate "GraphQL Book type" file distinct from the "Mongoose Book schema" file, they are one and the same. The tradeoff is the same kind of coupling the MongoDB notes call out with the `extends Document` pattern: `Book` is permanently tied to both Mongoose and GraphQL, you could not reuse this exact class definition somewhere that has nothing to do with either.

## extends Document, the same pattern as the MongoDB sibling project

```ts
export class Book extends Document {
```

Just like `User`, `Employee`, `Product`, and every other schema in the MongoDB sibling project except `Student`, `Book` extends Mongoose's own `Document` class directly, rather than using the `Book & Document` intersection style. That means any variable typed as `Book` already carries every method a real, saved Mongoose document has, `.save()` included, with no second type name needed anywhere in this codebase. This is exactly why `book.service.ts` can write `Model<Book>` directly (covered in [05-the-book-service.md](05-the-book-service.md)), there is no `BookDocument` type in this project at all, `Book` itself already plays that role.

## The _id field

```ts
   @Field(() => ID)
   declare readonly _id: string;
```

This field deserves a slow read because three different pieces of TypeScript and NestJS syntax are stacked into two lines. `declare` is a TypeScript keyword that tells the compiler "this property already exists on the base class, I am only re declaring its type here, do not emit any actual field initialization code for it." Since `Book extends Document`, and Mongoose's `Document` already defines a real `_id` property at runtime, `declare` avoids TypeScript generating a conflicting or redundant field. `readonly` is a plain TypeScript modifier meaning the field cannot be reassigned after the object is constructed, which makes sense for a database identifier, an existing document's id should never change out from under it. The type is given as `string` here for convenience, even though MongoDB's real `_id` values are `ObjectId` instances internally, Mongoose is flexible about this at the boundary and this project treats it as a string wherever it is exposed.

`@Field(() => ID)` is the GraphQL side of this same field. `ID` here is not the TypeScript type `string`, it is a special scalar type imported from `@nestjs/graphql`, GraphQL's own built in `ID` type, which exists specifically to represent unique identifiers. Serialized over the wire, a GraphQL `ID` looks exactly like a GraphQL `String`, both are quoted text, but marking a field as `ID` rather than `String` documents its intent clearly in the schema, this field is an identifier, not ordinary text data. The function syntax, `() => ID`, rather than just passing `ID` directly, is required by `@nestjs/graphql` here because `_id` is typed as a primitive `string` on the class, TypeScript's own reflection cannot tell the difference between "this is a `String` field" and "this is an `ID` field" just from the `string` type annotation, so the decorator needs to be told explicitly which GraphQL type to use, wrapped in a function so it can be evaluated lazily.

## The rest of the fields, Mongoose and GraphQL side by side

```ts
   @Prop({ required: true })
   @Field()
   title: string;

   @Prop()
   @Field({ nullable: true })
   description?: string;

   @Prop({ required: true })
   @Field()
   author: string
```

Each of these three fields carries both decorators stacked directly on top of each other, and the two decorators' options are kept consistent with each other by hand, which is worth noticing since nothing in the framework forces that consistency automatically. `title` and `author` both have `@Prop({ required: true })`, Mongoose's own validation rule meaning MongoDB will refuse to save a `Book` document missing either field, paired with a plain `@Field()` with no options, which in `@nestjs/graphql` means the field is non nullable in the generated schema by default, matching the Mongoose requirement. `description` has a bare `@Prop()` with no options, making it optional as far as MongoDB is concerned, paired with `@Field({ nullable: true })`, which tells the schema generator this field is allowed to be `null` or omitted in a GraphQL response, again matching. The TypeScript type for `description` also uses the optional property marker, `description?: string`, the same three way consistency (Mongoose optional, GraphQL nullable, TypeScript optional) that the MongoDB sibling project's notes point out has to be maintained by the developer rather than being enforced by any single source of truth.

## SchemaFactory.createForClass, exactly as in the MongoDB project

```ts
export const BookSchema = SchemaFactory.createForClass(Book);
```

This line works identically to every other schema file in the MongoDB sibling project, it reads all of the `@Prop()` metadata attached to `Book` and converts it into a real `mongoose.Schema` instance that Mongoose itself understands. The resulting `BookSchema` constant is what later gets registered with `MongooseModule.forFeature([{ name: Book.name, schema: BookSchema }])` inside `book.module.ts`, giving `BookService` an injectable Mongoose `Model` to work with, the exact same `forFeature`/`@InjectModel` pattern documented in the MongoDB project's own note on that topic. Notice that `@ObjectType()` plays no role at all in this line, `SchemaFactory.createForClass` only ever looks at Mongoose's own `@Prop()` metadata, the GraphQL decorators on the same class are entirely invisible to it, and vice versa, `@nestjs/graphql`'s schema builder only looks at `@ObjectType()`/`@Field()` metadata when it generates `schema.gql`, ignoring `@Prop()` and `@Schema()` completely. The two decorator sets coexist on one class without ever interfering with each other, because each library's tooling only reads the metadata keys that belong to it.
