# 02. The User Schema and Password Handling

## The whole schema, in full

This is the entire contents of `src/auth/user.schema.ts`:

```ts
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { Document } from 'mongoose';

export type UserDocument = User & Document;

@Schema()
export class User {
    @Prop({ required: true, unique: true })
    email: string;

    @Prop({required: true})
    password: string
}

export const UserSchema = SchemaFactory.createForClass(User);
```

There are exactly two stored fields on a `User` document, `email` and `password`. That is the entire data model this authentication system needs, this project does not track a name, a role, a signup date, or anything else about a user beyond the two things required to authenticate them.

`@Schema()` is used here with no options passed in, which means, notably, there is no `timestamps: true`. Unlike some of the schemas in the sibling MongoDB project, a `User` document here will not automatically get `createdAt` or `updatedAt` fields, MongoDB will simply store whatever this schema defines and nothing more (plus the `_id` field every MongoDB document gets automatically, and the `__v` version key Mongoose adds by default).

`@Prop({ required: true, unique: true })` on `email` does two separate things. `required: true` is a Mongoose validation rule, it means `.save()` will reject a document that has no `email` value at all, throwing a validation error before anything reaches the database. `unique: true` is a different mechanism entirely, it tells Mongoose to build a unique index on the `email` field in MongoDB itself, so the database engine, not just the application, will refuse to store two documents with the same email value. This is the mechanism that stops the same email address from being used to register two separate accounts. It is worth being precise about what `unique: true` actually is: it is not a validation rule checked before save the way `required` is, it is an index constraint enforced by MongoDB when the write actually happens, which means a duplicate email attempt does not fail gracefully with a nice Mongoose validation error, it fails with a raw MongoDB duplicate key error instead. What that means for the actual `signup()` flow is covered in the next note, since this project does not currently catch that error anywhere.

`@Prop({required: true})` on `password` marks it as a required stored field with no other options, no `unique`, no `select: false` to hide it from query results by default, nothing. That last point matters: since `password` has no `select: false`, any query that fetches a `User` document, including the one used during login, will include the stored password hash in the result by default, exactly the behavior the `login()` method in `auth.service.ts` actually depends on, since it needs `user.password` to compare against. The tradeoff is that if any other part of this codebase ever queried users for a different reason, the hash would come back in that result too unless the query explicitly excluded it, though as it stands there is only one place in this small project that reads a `User` document at all.

## Where password hashing does not happen

It is worth stating plainly, because it is easy to assume otherwise if you have seen other Mongoose authentication tutorials: there is no `pre('save', ...)` hook anywhere in `user.schema.ts`, and no hashing logic of any kind lives on the schema. A common pattern in Mongoose based authentication code is to attach a pre save middleware function directly to the schema, so that any code that saves a `User` document automatically gets its password hashed on the way in, no matter where the save call originates. That pattern is not used here. This schema is purely a data shape declaration, two required fields and a unique constraint, nothing else.

## Where password hashing actually happens instead

The hashing work is done entirely inside `AuthService.signup()`, in `src/auth/auth.service.ts`, using the `bcrypt` package directly at the point where a new user is created:

```ts
async signup(email: string, password: string){
    const hash = await bcrypt.hash(password, 10);
    const user = new this.userModel({ email, password: hash });
    return user.save();
}
```

`bcrypt.hash(password, 10)` takes the plain text password the client submitted and produces a salted hash, the `10` is the cost factor (also called the number of salt rounds), a higher number makes each hash slower to compute and correspondingly slower for an attacker to brute force, `10` is a commonly used, reasonable middle ground default. The important structural point is that the hashing happens explicitly, by name, inside the service method, immediately before a new `User` model instance is constructed with the already hashed value in its `password` field. By the time `new this.userModel({...})` runs, the plain text password the client sent has already been thrown away, only the hash is ever handed to Mongoose, and only the hash is ever written to MongoDB.

This has a direct, practical consequence worth calling out: because hashing lives in the service and not in the schema, it only happens if code goes through `AuthService.signup()`. If any other code path in this project (or a future one built on top of it) ever created a `User` document directly through `this.userModel.create(...)` or `new this.userModel(...)` without first calling `bcrypt.hash()` itself, that user's password would be stored in MongoDB completely in plain text, with nothing at the schema level to prevent it. As it stands today, `signup()` is the only place in this codebase that ever creates a `User` document, so this is not an active bug, but it is a real structural fragility worth understanding: the schema itself provides no safety net, the guarantee that passwords are always hashed rests entirely on every future caller remembering to hash before saving, rather than being enforced once, centrally, on the schema.

## `UserDocument`, the same intersection pattern used elsewhere

```ts
export type UserDocument = User & Document;
```

This is the exact same typing pattern you would see in the `Student & Document` example from the sibling MongoDB project's notes: `User` is the plain class describing the two stored fields, and Mongoose's own `Document` type is a large interface describing everything a real, saved MongoDB document has, `.save()`, `._id`, and so on. Intersecting the two gives you a type that has both. This `UserDocument` type is what shows up as the generic argument in `Model<UserDocument>` inside `AuthService`, giving full autocomplete and type checking on every Mongoose method called against `this.userModel` throughout that class.
