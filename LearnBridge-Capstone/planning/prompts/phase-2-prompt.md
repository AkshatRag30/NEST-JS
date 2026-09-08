# Phase 2 Prompt: Authentication and Users

Use this once phase 1 is complete, verified, and your app boots successfully against a real local Postgres and MongoDB connection with a real `.env` file in place.

Copy everything inside the fenced block below into Claude Code as one message.

```
Before writing any code, read these files completely: planning/04-data-model-and-relationships.md (specifically the User section) and planning/06-phase-2-PRD-auth-and-users.md. This is phase 2, building directly on the phase 1 foundation that already exists in this project, do not modify the phase 1 config, logging, filter, or pipe setup unless something about it is actually broken. Do not build courses, categories, enrollments, or anything else from a later phase yet.

Add a User model to prisma/schema.prisma with an id (use a uuid default), a unique email, a passwordHash field, a name field, a role field using a Prisma enum with the values STUDENT, INSTRUCTOR, and ADMIN, and a createdAt field defaulting to now. Run a Prisma migration to apply this.

Install @nestjs/jwt, @nestjs/passport, passport, passport-jwt, bcrypt, and the matching @types packages for passport-jwt and bcrypt as dev dependencies.

Add JWT_EXPIRES_IN as an additional, optional environment variable (default it to '1d' in code if it is not set, do not add it to the required validation list from phase 1 since it has a safe default).

Build a UsersModule with a service that can find a user by email, find a user by id, and create a user, all through PrismaService, and a controller exposing GET /users/me, protected by a JWT auth guard, returning the authenticated user's own record with the passwordHash field stripped out before it is returned, never send a password hash back to a client under any circumstance.

Build an AuthModule with a SignupDto (email validated as a real email, password with a minimum length of 8, name required, and an optional role restricted to only STUDENT or INSTRUCTOR, never ADMIN, defaulting to STUDENT if omitted) and a LoginDto (email and password). Build an AuthService with a signup method that first checks whether a user with that email already exists and throws a ConflictException with a clear message if so, before doing anything else, then hashes the password with bcrypt using a cost factor of 10, creates the user, and returns the created user with the passwordHash stripped out. Build a login method that looks up the user by email, and if no user is found or bcrypt.compare on the password fails, throws a genuine UnauthorizedException with the message 'Invalid credentials', it must never return null or any kind of success response on bad credentials. On successful login, sign a JWT with JwtService containing the user's id and role in the payload, using the expiry from configuration, and return it as an accessToken.

Build a JwtStrategy extending PassportStrategy(Strategy, 'jwt'), reading the secret through ConfigService, extracting the token from the Authorization bearer header, not accepting an expired token, and returning an object containing at least the user's id and role from the payload in its validate method, this becomes req.user on any guarded route. Build a JwtAuthGuard extending AuthGuard('jwt'). Build a RolesGuard that reads role metadata set by a custom Roles decorator using Reflector, comparing it against req.user.role, and throwing a ForbiddenException on a mismatch, along with the Roles decorator itself built with SetMetadata. Do not apply the RolesGuard anywhere yet beyond a placeholder demonstration if you want one, its real use starts in phase 3 when instructor only routes exist.

Build an AuthController exposing POST /auth/signup and POST /auth/login, wired to the DTOs and service methods above.

Write focused unit tests for AuthService only, covering three cases, signup with an email that already exists throws ConflictException, login with a wrong password throws UnauthorizedException, and login with correct credentials returns an object containing an accessToken. Mock PrismaService and JwtService by hand, do not use a real database or a bare empty providers array.

When you are done, tell me the exact request bodies for signing up a student and logging in, and confirm out loud that a failed login in your own test actually returns a 401 status, not a 200 with an empty or null body.
```
