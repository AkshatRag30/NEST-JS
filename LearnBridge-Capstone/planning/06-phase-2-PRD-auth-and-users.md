# Phase 2 PRD: Authentication and Users

## Goal

Build real account creation, login, and route protection, adapting the pattern already proven correct in `JWT-Auth-with-Mongo-DB-Nest-JS-main` onto a Prisma backed `User` table instead of a Mongoose schema, and fixing the two real bugs found in that reference project's login and signup flow along the way.

## Concepts practiced

DTOs and `class-validator`, `bcrypt` password hashing, `@nestjs/jwt` token signing, `passport-jwt` and `@nestjs/passport` strategy validation, guards and `CanActivate`, a custom roles guard built with `Reflector` and a custom `@Roles()` decorator, and the authentication versus authorization distinction. See [03-concept-coverage-map.md](03-concept-coverage-map.md) for the exact source notes behind each of these.

## Scope

A `UsersModule` owning the Prisma backed `User` model, with a service method to find a user by email and a service method to create one. An `AuthModule` with a `signup` endpoint accepting email, password, name, and role (restricted to STUDENT or INSTRUCTOR, never ADMIN, since an admin account is seeded directly rather than self registered), which must check for an existing email first and throw a `ConflictException` with a clear message if one exists, rather than letting a raw database unique constraint error bubble up as an unhandled 500, which is exactly what happened in the reference project. The password is hashed with `bcrypt.hash` before being saved, and the hash never appears in any response returned to the client.

A `login` endpoint accepting email and password, which fetches the user, compares the password with `bcrypt.compare`, and on any failure, wrong email or wrong password, throws a genuine `UnauthorizedException`, returning a real 401, not the plain `null` with a 200 status the reference project's login method returned on bad credentials. On success it signs a JWT with `JwtService.sign()`, with a payload containing the user's id and role, and an expiry, read from `ConfigService`, not hardcoded.

A `JwtStrategy` extending `PassportStrategy(Strategy)`, reading the secret through `ConfigService`, extracting the token from the `Authorization` bearer header, and returning the decoded payload from `validate()`, which becomes `req.user` on any route protected by it. A `JwtAuthGuard` extending `AuthGuard('jwt')`. A `RolesGuard` reading a `@Roles(...)` decorator's metadata off the route handler with `Reflector`, and comparing it against `req.user.role`, throwing `ForbiddenException` on a mismatch. Both guards get applied deliberately, route by route, with an explicit note in each later phase's PRD about which roles can call which endpoint, so that unlike the Supabase guard sitting unused in the PostgreSQL reference project, no guard in LearnBridge is ever defined without also being applied somewhere real.

## API surface

`POST /auth/signup`, `POST /auth/login`, and `GET /users/me`, a protected route returning the currently authenticated user's own profile, which exists specifically to give you one simple route to prove the whole guard and strategy chain actually works before building anything more complex on top of it.

## Acceptance criteria

1. Signing up twice with the same email returns a 409 with a clear message the second time, not a 500.
2. Logging in with a wrong password returns a real 401, never a 200 with an empty body.
3. A successful login returns a JWT that, when decoded by hand (for example at jwt.io), contains the user's id and role and nothing sensitive like the password hash.
4. Calling `GET /users/me` with no token, or an expired or tampered token, returns a 401.
5. Calling `GET /users/me` with a valid token returns exactly that user's own profile, with no password hash in the response.
6. A `@Roles('INSTRUCTOR')` protected route, once one exists in phase 3, rejects a valid token belonging to a STUDENT with a 403, proving the roles guard checks the actual token's role and not just whether a token exists at all.

## Explicit trap to avoid

Do not return `null` or an empty success response for any authentication failure. Every failure path must throw the correct, named Nest exception, so the HTTP status code the client sees actually matches what happened.
