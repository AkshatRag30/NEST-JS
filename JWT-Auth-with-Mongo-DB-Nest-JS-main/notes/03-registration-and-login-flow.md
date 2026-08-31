# 03. The Registration and Login Flow

## `AuthService` in full

Here is the entire `src/auth/auth.service.ts`, worth reading top to bottom once before breaking it apart:

```ts
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { User, UserDocument } from './user.schema';
import { JwtService } from '@nestjs/jwt';
import { Model } from 'mongoose';
import * as bcrypt from 'bcrypt';

@Injectable()
export class AuthService {
    constructor(
        @InjectModel(User.name) private userModel: Model<UserDocument>, private jwtService: JwtService,
    ) {}

    async signup(email: string, password: string){
        const hash = await bcrypt.hash(password, 10);
        const user = new this.userModel({ email, password: hash });
        return user.save();
    }
    async login(email: string, password: string){
        const user = await this.userModel.findOne({ email });
        if(!user) return null;
        const isMatch = await bcrypt.compare(password, user.password);
        if(!isMatch) return null;
        const payload = { email: user.email, sub: user._id };
        return {
            access_token: this.jwtService.sign(payload),
        }
    }

}
```

## The constructor: two injected dependencies

`@InjectModel(User.name) private userModel: Model<UserDocument>` is the same Mongoose model injection pattern covered for the sibling MongoDB project, it hands `AuthService` a queryable, saveable `Model` object tied to the `User` schema, made possible because `AuthModule` registered that schema with `MongooseModule.forFeature([{ name: User.name, schema: UserSchema }])`.

`private jwtService: JwtService` is injected the ordinary way, by its class type, no special decorator needed, because `JwtService` is a normal class based provider. It becomes available here because `AuthModule` imports `JwtModule.registerAsync({...})` (covered fully in the next note), and any module that imports `JwtModule` gets `JwtService` available for injection into its own providers. `JwtService` is the small wrapper `@nestjs/jwt` provides around the underlying `jsonwebtoken` library, giving you `.sign()` to create a token and `.verify()` to check one, without having to call `jsonwebtoken` directly.

## `signup()`, step by step

```ts
async signup(email: string, password: string){
    const hash = await bcrypt.hash(password, 10);
    const user = new this.userModel({ email, password: hash });
    return user.save();
}
```

The method receives a plain email and password, both plain strings, straight from the controller, with no validation applied to either before this point (more on that in a moment). `bcrypt.hash(password, 10)` hashes the plain text password with a cost factor of `10` salt rounds, as covered in the previous note. `new this.userModel({ email, password: hash })` constructs a new, unsaved Mongoose document in memory, using the object shorthand for `{ email: email, password: hash }`. `user.save()` is what actually performs the write to MongoDB, returning a promise that resolves to the saved document once it succeeds, and `signup()` returns that promise directly, so whatever `.save()` resolves to becomes the HTTP response body for this route (see [06-auth-controller-routes.md](06-auth-controller-routes.md) for what a client actually receives).

There are two real, honest gaps worth calling out plainly here, since this project has no validation layer catching them earlier.

First, `signup()` never checks whether an account with this email already exists before attempting to save. It relies entirely on the `unique: true` index on `email` (from the previous note) to reject the second registration at the database level. That means a duplicate signup attempt does not fail with a clean, friendly error, it fails because MongoDB itself throws a duplicate key error (an error with code `11000`) when `.save()` tries to write it, and nothing in `signup()` or in `AuthController` catches that error. Left uncaught, it propagates up as an unhandled rejection that Nest's default exception handling turns into a generic `500 Internal Server Error`, rather than a clear `409 Conflict` telling the client the email is already taken.

Second, `signup()` returns the entire saved Mongoose document as the response body, `return user.save();`, with no step that strips the `password` field out first. Since `password` has no `select: false` on it in the schema (see the previous note), the response a client receives after signing up includes their own bcrypt hash sitting right there in the JSON body, alongside `_id`, `email`, and `__v`. It is not the plain text password, bcrypt hashes are specifically designed to be safe even if exposed, but returning any form of the password field back to the client in an API response is still a real, avoidable habit to flag, a production version of this code would map the saved document to a plain object containing only `email` and `_id` before returning it.

Third, and worth noting because it compounds the first two gaps, there is no validation at all on the shape of the incoming request body before it reaches `signup()`. As covered in [06-auth-controller-routes.md](06-auth-controller-routes.md), the controller types the body as `{email: string; password: string}` purely as a TypeScript annotation, which has no effect at runtime, and `main.ts` never registers a global `ValidationPipe`. A request with a missing or empty `password` field will sail straight through into `bcrypt.hash(undefined, 10)`, and depending on what value actually arrives, this either throws inside `bcrypt` or produces a document that then fails Mongoose's own `required: true` check on `password` at save time, again surfacing to the client as an unhandled `500` rather than a clean `400 Bad Request`.

## `login()`, step by step

```ts
async login(email: string, password: string){
    const user = await this.userModel.findOne({ email });
    if(!user) return null;
    const isMatch = await bcrypt.compare(password, user.password);
    if(!isMatch) return null;
    const payload = { email: user.email, sub: user._id };
    return {
        access_token: this.jwtService.sign(payload),
    }
}
```

`this.userModel.findOne({ email })` looks up a single `User` document by email, exactly like any other Mongoose query. If no document matches, `user` is `null`, and `login()` immediately returns `null` itself.

If a user was found, `bcrypt.compare(password, user.password)` is the exact mechanism by which a password gets checked against its hash. It is critical to understand that this is not a matter of hashing the incoming password again and comparing the two hash strings with `===`, bcrypt hashes embed their own salt directly inside the stored hash string, and `bcrypt.compare()` knows how to extract that salt, hash the freshly supplied plain text password with it internally, and compare the results in a way that resists timing attacks. `isMatch` is a boolean, and if it is `false`, `login()` again returns `null`, exactly the same return value as the "no such user" case above.

This is a real, honest gap worth calling out clearly: `login()` gives the exact same response, a bare `null`, whether the email does not exist at all or the email exists but the password is wrong, which is actually a reasonable security choice in principle (not revealing which of the two failed avoids leaking which emails are registered), but the mechanism used to express that here is not appropriate for an HTTP API. Returning plain `null` from a NestJS controller method results in an HTTP response with status `200 OK` and a body of `null`, not a `401 Unauthorized`. A client cannot tell the difference between "login failed" and "login succeeded and the server genuinely has nothing to say," except by inspecting the body shape after the fact. A correct implementation would throw a NestJS `UnauthorizedException` in both failure branches, which Nest's built in exception handling automatically turns into a proper `401` response, but that is not what this code does.

If both checks pass, `const payload = { email: user.email, sub: user._id };` builds the data that will be embedded inside the JWT, and `this.jwtService.sign(payload)` produces the signed token string, returned to the client wrapped in `{ access_token: ... }`. Exactly how that signing step works, where the secret comes from, and what `sub` means, is covered in full in the next note.
