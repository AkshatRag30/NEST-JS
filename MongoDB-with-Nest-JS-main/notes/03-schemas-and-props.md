# 03. Schemas and Props

Every one of the six feature modules in this repo defines at least one Mongoose schema using the same three building blocks from `@nestjs/mongoose`: the `@Schema()` class decorator, the `@Prop()` property decorator, and the `SchemaFactory.createForClass()` function. This note explains all three using the simplest example in the repo, `src/student/student.schema.ts`, and then explains a second pattern used everywhere else in the repo that looks different but does almost the same job.

## The plainest schema in the repo

```ts
import { Prop, Schema, SchemaFactory } from "@nestjs/mongoose";
import { Document } from "mongoose";

export type StudentDocument = Student & Document;

@Schema({ timestamps: true })
export class Student {
    @Prop({ required: true })
    name: string;

    @Prop({ required: true })
    age: number;

    @Prop()
    email?: string;
}

export const StudentSchema = SchemaFactory.createForClass(Student);
```

`@Schema({ timestamps: true })` marks the `Student` class as a Mongoose schema definition. The `timestamps: true` option is a Mongoose feature, not something `@nestjs/mongoose` invents, it tells Mongoose to automatically add and maintain two extra fields on every document of this type, `createdAt` and `updatedAt`, filled in and updated automatically every time a document is saved. Neither field appears anywhere in the `Student` class itself, they are added behind the scenes by Mongoose at the schema level.

`@Prop({ required: true })` on `name` and `age` marks each of them as an actual field that gets stored in MongoDB, with a validation rule attached, `required: true` means Mongoose will refuse to save a student document that is missing that field, throwing a validation error instead. `@Prop()` on `email`, with no options, still marks it as a stored field, but with no validation constraints, so it is optional. Notice the TypeScript type also reflects that: `email?: string` uses TypeScript's own optional property syntax, this is purely a TypeScript compile time hint (it does not affect Mongoose's runtime validation on its own), but the fact that `email` has no `required: true` in its `@Prop()` is what actually makes it optional as far as MongoDB is concerned.

`SchemaFactory.createForClass(Student)` is the step that converts this decorated class into an actual `mongoose.Schema` instance, the object type Mongoose itself understands, by reading all the metadata that `@Prop()` attached to the class under the hood using `reflect-metadata`. The resulting `StudentSchema` constant is what later gets registered with Mongoose in `student.module.ts` (covered in the next note).

## `StudentDocument`, a document type built with an intersection

```ts
export type StudentDocument = Student & Document;
```

This line defines a new TypeScript type, not a class, using the `&` intersection operator. `Student` is the plain data shape (`name`, `age`, `email`), and Mongoose's own `Document` type is a large interface describing every method and field a real, saved MongoDB document has, things like `.save()`, `._id`, `.isNew`, and dozens of others that a plain class does not have. Intersecting the two types produces a type that TypeScript treats as having both sets of members at once. This `StudentDocument` type is what gets used everywhere the code needs to talk about "a real Mongoose document that also happens to have the `name`/`age`/`email` shape", most importantly in `student.service.ts`'s `Model<StudentDocument>` (see [04-forfeature-and-dependency-injection.md](04-forfeature-and-dependency-injection.md)).

## The other pattern used in every other module: `extends Document`

Every other schema in this repo, `User`, `Address`, `Employee`, `Profile`, `Product`, `Tag`, `Library`, `Book`, `Project`, and `Developer`, uses a different approach:

```ts
// src/user/schemas/user.schema.ts
@Schema()
export class User extends Document {
    @Prop()
    name: string;

    @Prop({ type: Address })
    address: Address
}
```

Instead of defining a separate `UserDocument` type via intersection, `User` itself directly extends Mongoose's `Document` class. This means any variable typed as `User` already has both the `name`/`address` fields and every method `Document` provides, `.save()` included, with no second type needed. That is why in `user.service.ts` you will see `Model<User>` used directly, there is no separate "document" type name at all in that module.

Both approaches accomplish the same goal: giving TypeScript a type that represents "this class's fields, plus everything a real saved Mongoose document has." The `Student & Document` intersection style is generally considered the more idiomatic `@nestjs/mongoose` pattern (it is the one shown in NestJS's own official documentation), because it keeps the plain data class (`Student`) free of any Mongoose specific inheritance, which matters if you ever want to reuse that same class shape somewhere that has nothing to do with Mongoose, such as a DTO or a plain API response type. The `extends Document` style used in the rest of this repo is slightly shorter to write but permanently couples the class itself to Mongoose. Seeing both patterns side by side in the same codebase is a useful, honest reminder that this repo was built incrementally as a set of small teaching examples rather than as one project with one enforced convention throughout.

## Sub-schemas without their own document type: `Address` and `Tag`

```ts
// src/user/schemas/address.schema.ts
@Schema()
export class Address {
    @Prop()
    street: string;

    @Prop()
    city: string;
}
```

`Address` and, similarly, `Tag` in the product module, are also decorated with `@Schema()` and `@Prop()`, but neither one extends `Document`, and neither one gets a `SchemaFactory.createForClass()` call of its own, in fact `Address` does not even export a schema constant at all (only the class). That is intentional: these two classes are never saved to MongoDB as their own, independent documents in their own collection. They exist purely to describe the shape of data that gets embedded inside another document (`User.address` and `Product.tags`). This distinction, between a schema that becomes its own MongoDB collection and a schema that only ever exists nested inside another document, is exactly the "embedding" concept explained fully in [06-embedding-subdocuments.md](06-embedding-subdocuments.md).
