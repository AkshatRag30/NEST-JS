# JWT Auth with MongoDB and NestJS — Study Guide

This folder holds a complete set of topic by topic notes built from this repository. Unlike the MongoDB relationships project sitting next to it, this project is not about modeling data shapes, it is about one specific, very common backend problem: how does a user register an account, log in with a password, and then prove on every later request that they are still that same logged in user, without the server having to store any session state. The answer this project implements is JSON Web Tokens, wired into NestJS through `@nestjs/jwt` and `@nestjs/passport`, backed by a single `User` collection in MongoDB through `@nestjs/mongoose`.

This is a small project, only one real feature module (`auth`), one schema (`User`), one service, one controller, and one Passport strategy. That smallness is deliberate. Every note below reads the actual code line by line rather than describing JWT authentication in the abstract, so you can see exactly how little code it actually takes to get real, working token based authentication running, and exactly where this particular implementation cuts corners a production app would not.

## How to read these notes

Go in order the first time. Each note builds on the one before it.

1. [01-project-setup-and-mongodb-connection.md](01-project-setup-and-mongodb-connection.md) — what this project is, the real dependencies in `package.json`, and how `app.module.ts` connects to MongoDB and wires in `AuthModule`.
2. [02-user-schema-and-password-handling.md](02-user-schema-and-password-handling.md) — `user.schema.ts`, the two stored fields, the unique email constraint, and exactly where password hashing actually happens (it is not in the schema).
3. [03-registration-and-login-flow.md](03-registration-and-login-flow.md) — `auth.service.ts`'s `signup` and `login` methods, read in full, including the honest gaps in how failure is handled.
4. [04-jwt-signing-and-token-payload.md](04-jwt-signing-and-token-payload.md) — how `JwtModule.registerAsync` is configured, where the signing secret actually comes from, what goes into the token payload, and what expiry is set.
5. [05-passport-jwt-strategy-and-protected-routes.md](05-passport-jwt-strategy-and-protected-routes.md) — `jwt.strategy.ts`, how it extends `PassportStrategy`, how a token is pulled off the request and verified, what `validate()` returns, and what that becomes on `req.user`.
6. [06-auth-controller-routes.md](06-auth-controller-routes.md) — the three routes this project exposes, `POST /auth/signup`, `POST /auth/login`, and the guarded `GET /auth/profile`.
7. [07-testing-gaps.md](07-testing-gaps.md) — an honest look at all four `.spec.ts` files in this repo, none of which will pass if you actually run them, and each for a slightly different reason.
8. [08-full-recap-and-file-map.md](08-full-recap-and-file-map.md) — tracing one real user's journey, signing up, logging in, and then calling the protected profile route, through every layer of this codebase, plus a table mapping every file to the note that explains it.

## A note on the source material

There are no slides or lecture PDFs in this repository, only working source code. Every claim in these notes was checked directly against the actual file it describes rather than assumed from how a typical JWT tutorial usually looks, because this project does deviate from the typical tutorial in a few small but real ways, and those differences are called out honestly rather than smoothed over. This is a genuinely working authentication implementation, registration, password hashing, login, and token verification all actually function, so the goal of these notes is to explain the real mechanics in depth, not to hunt for missing functionality.
