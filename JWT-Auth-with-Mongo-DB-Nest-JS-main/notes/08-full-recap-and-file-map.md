# 08. Full Recap: One User, Start to Finish

This last note traces one real user's full journey, signing up, logging in, and then calling the one protected route, through every layer covered in this folder, and then gives you a single table mapping every file in this repository to the note that explains it, so this folder can double as a quick reference after your first read through.

## Step one: `POST /auth/signup`

A client sends `POST /auth/signup` with a JSON body of `{ "email": "person@example.com", "password": "correct-horse" }`. Nest's router, built when `NestFactory.create(AppModule)` ran at startup (see [01](01-project-setup-and-mongodb-connection.md)), matches this to `AuthController.signup`, because `@Controller('auth')` combined with `@Post('signup')` on that method is exactly `POST /auth/signup`.

`@Body() body` receives the parsed request body with no validation applied to it (see [06](06-auth-controller-routes.md)), and `signup()` immediately forwards `body.email` and `body.password` into `AuthService.signup()`. Inside that method, `bcrypt.hash(password, 10)` (see [02](02-user-schema-and-password-handling.md)) turns the plain text password into a salted hash, `new this.userModel({ email, password: hash })` builds a new in memory `User` document using the `Model<UserDocument>` that was injected via `@InjectModel(User.name)`, made possible because `AuthModule` registered `User` and `UserSchema` with `MongooseModule.forFeature([...])`. `user.save()` writes that document into the `users` collection on the MongoDB instance `MongooseModule.forRoot(process.env.MONGO_URI!)` connected to back when the app started (see [01](01-project-setup-and-mongodb-connection.md)), with MongoDB enforcing the unique index on `email` at the moment of the write. The saved document, hash and all, travels back up through `AuthService` and `AuthController`, and Nest's built in response handling serializes it into the JSON response, no manual `res.json(...)` call anywhere in this codebase.

## Step two: `POST /auth/login`

The same user later sends `POST /auth/login` with the same email and password. `AuthController.login` forwards both into `AuthService.login()`. `this.userModel.findOne({ email })` fetches the just created document back out of MongoDB, including its hashed password, since nothing in the schema hides that field from query results (see [02](02-user-schema-and-password-handling.md)). `bcrypt.compare(password, user.password)` checks the freshly submitted plain text password against the stored hash without ever needing to reverse the hash itself (see [03](03-registration-and-login-flow.md)). Assuming it matches, `const payload = { email: user.email, sub: user._id }` builds the data going into the token, and `this.jwtService.sign(payload)` produces a signed JWT, using the secret and the one hour expiry configured back in `AuthModule`'s `JwtModule.registerAsync({...})` (see [04](04-jwt-signing-and-token-payload.md)). The client receives `{ "access_token": "eyJhbGciOi..." }` as the response body.

## Step three: `GET /auth/profile`, with the token in hand

The client now sends `GET /auth/profile`, this time attaching a header, `Authorization: Bearer eyJhbGciOi...`, using the token from the previous step. Because this route carries `@UseGuards(AuthGuard('jwt'))`, NestJS runs the guard before `getProfile()` itself executes. `AuthGuard('jwt')` activates the strategy registered under the name `'jwt'`, which is `JwtStrategy` (see [05](05-passport-jwt-strategy-and-protected-routes.md)). Inside that strategy, `ExtractJwt.fromAuthHeaderAsBearerToken()` pulls the raw token string out of the `Authorization` header, and `passport-jwt` verifies its signature against the same `MY_JWT_SECRET` read through `ConfigService`, also confirming it has not yet passed its one hour expiry. Once verified, the decoded payload, `{ email, sub, iat, exp }`, is handed to `JwtStrategy.validate()`, which returns `{ userId: payload.sub, email: payload.email }`, and NestJS attaches that exact object to `req.user`.

Only now does `getProfile(@Request() req)` actually run, and its entire body is `return req.user;`. The client receives `{ "userId": "<the same MongoDB _id that was in sub>", "email": "person@example.com" }` as the response. No second database query happened at this step, the entire identity check rested on the cryptographic signature check against the shared secret, exactly the point of using a stateless, signed token instead of a server side session.

If, instead, the client had sent no `Authorization` header at all, an expired token, or a token signed with a different secret, the guard would have rejected the request with `401 Unauthorized` before `getProfile()` ever ran, and none of `AuthController`'s code for that route would have executed.

## File map

| File | What it teaches | Note |
|---|---|---|
| `package.json` | The real dependencies, `@nestjs/jwt`, `@nestjs/passport`, `passport-jwt`, `bcrypt`, `@nestjs/mongoose`, `mongoose`, `@nestjs/config`, and their exact versions | [01](01-project-setup-and-mongodb-connection.md) |
| `src/main.ts` | App bootstrap, `NestFactory.create`, `app.listen`, and the absence of a global `ValidationPipe` | [01](01-project-setup-and-mongodb-connection.md) |
| `src/app.module.ts` | `ConfigModule.forRoot({ isGlobal: true })`, `MongooseModule.forRoot(process.env.MONGO_URI!)`, wiring in `AuthModule` | [01](01-project-setup-and-mongodb-connection.md) |
| `src/app.controller.ts`, `src/app.service.ts` | The untouched Nest CLI starter controller and service, the only unguarded route in the app | [01](01-project-setup-and-mongodb-connection.md) |
| `src/auth/user.schema.ts` | The `User` schema, `email` and `password` fields, the unique index on `email`, and confirmation that hashing does not happen here | [02](02-user-schema-and-password-handling.md) |
| `src/auth/auth.module.ts` | `MongooseModule.forFeature` for `User`, `JwtModule.registerAsync`, the `AuthService`/`JwtStrategy` providers | [02](02-user-schema-and-password-handling.md), [04](04-jwt-signing-and-token-payload.md) |
| `src/auth/auth.service.ts` | `signup()` and `login()` in full, `bcrypt.hash`, `bcrypt.compare`, the JWT payload construction, and the honest gaps around error handling and response shape | [03](03-registration-and-login-flow.md), [04](04-jwt-signing-and-token-payload.md) |
| `src/auth/jwt.strategy.ts` | `PassportStrategy(Strategy)`, `ExtractJwt.fromAuthHeaderAsBearerToken()`, where the secret comes from, `validate()` and `req.user` | [05](05-passport-jwt-strategy-and-protected-routes.md) |
| `src/auth/auth.controller.ts` | The three routes, `POST /auth/signup`, `POST /auth/login`, and the guarded `GET /auth/profile` | [06](06-auth-controller-routes.md) |
| `src/app.controller.spec.ts` | The one spec file in this repo that actually passes | [07](07-testing-gaps.md) |
| `src/auth/auth.controller.spec.ts`, `src/auth/auth.service.spec.ts` | Scaffold tests that fail once real constructor dependencies exist and are never supplied | [07](07-testing-gaps.md) |
| `src/auth/user.schema.spec.ts` | A test that calls `new` on a compiled Mongoose schema instance, which is not a constructor | [07](07-testing-gaps.md) |
| `test/app.e2e-spec.ts` | End to end test, and why it needs both a real `MONGO_URI` and a real `MY_JWT_SECRET` to pass | [07](07-testing-gaps.md) |

## The single idea underneath this whole project

If you take away one thing from this folder, it should be this: JWT authentication, at its core, is just two matching halves of the same secret being used at two different moments. `login()` uses that secret to produce a signature nobody without the secret could forge, and `JwtStrategy` uses the exact same secret, later, on a completely different request, to check whether a presented signature could only have been produced by someone who had that secret. Everything else in this project, the bcrypt hashing, the Mongoose schema, the guard, the payload shape, exists to support that one core exchange: prove once with a password, receive a signed claim, and present that claim instead of the password on every request after that, until it expires.
