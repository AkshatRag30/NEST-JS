# 06. The Auth Controller's Routes

## `auth.controller.ts` in full

```ts
import { Body, Controller, Get, Post, Request, UseGuards } from '@nestjs/common';
import { AuthService } from './auth.service';
import { AuthGuard } from '@nestjs/passport';

@Controller('auth')
export class AuthController {
    constructor(private authService: AuthService){}

    @Post('signup')
    signup(@Body() body: {email: string; password: string}){
        return this.authService.signup(body.email, body.password);
    }
    @Post('login')
    login(@Body() body: {email: string; password: string}){
        return this.authService.login(body.email, body.password);
    }
    @UseGuards(AuthGuard('jwt'))
    @Get('profile')
    getProfile(@Request() req){
        return req.user;
    }
}
```

`@Controller('auth')` prefixes every route in this class with `/auth`, so the three methods below map to `POST /auth/signup`, `POST /auth/login`, and `GET /auth/profile`. The constructor injects `AuthService` the ordinary way, and every method in this controller is a thin wrapper, none of them contain any real logic themselves, all of the actual work happens in the service methods covered in earlier notes.

## `POST /auth/signup`

```ts
@Post('signup')
signup(@Body() body: {email: string; password: string}){
    return this.authService.signup(body.email, body.password);
}
```

`@Post('signup')` combined with the controller's `'auth'` prefix means this handles `POST /auth/signup`. `@Body() body: {email: string; password: string}` pulls the parsed JSON request body into `body`, typed inline as an object with `email` and `password` string fields. It is worth being precise about what that inline type annotation actually does and does not do: it is purely a TypeScript compile time hint for whoever is reading or writing this code, it has no effect whatsoever at runtime. There is no DTO class here decorated with `class-validator` decorators, and as noted in [01](01-project-setup-and-mongodb-connection.md), there is no global `ValidationPipe` registered in `main.ts` either. That means whatever JSON body a client actually sends arrives at `authService.signup(...)` completely unchecked, if `email` or `password` is missing, empty, the wrong type, or simply absent from the request body entirely, nothing in this controller catches that before it reaches the service, where it either gets rejected by Mongoose's own schema level `required: true` validation (surfacing as a raw, unfriendly `500` response, as covered in [03](03-registration-and-login-flow.md)), or in some cases throws inside `bcrypt.hash()` itself.

The method simply forwards both fields straight into `AuthService.signup()` and returns whatever that returns, which, per the previous notes, is the full saved Mongoose document, hashed password field and all.

## `POST /auth/login`

```ts
@Post('login')
login(@Body() body: {email: string; password: string}){
    return this.authService.login(body.email, body.password);
}
```

Structurally identical to `signup()`, just routed to `/auth/login` and forwarding to `AuthService.login()` instead. The same lack of request validation applies here too. And as covered in [03](03-registration-and-login-flow.md), whatever `login()` resolves to becomes the response body directly, meaning a failed login (wrong email, or wrong password) currently results in an HTTP `200 OK` response carrying a JSON body of `null`, rather than a `401 Unauthorized`, since nothing in this controller inspects the service's return value before sending it back.

## `GET /auth/profile`, the one protected route

```ts
@UseGuards(AuthGuard('jwt'))
@Get('profile')
getProfile(@Request() req){
    return req.user;
}
```

This is the only route in the entire project guarded by anything. `AuthGuard('jwt')` is a function imported directly from `@nestjs/passport` itself (not a custom guard class this project defines, unlike some other NestJS tutorials that wrap it in a dedicated `JwtAuthGuard extends AuthGuard('jwt')` file for reuse or extension, this project simply calls the generic helper inline). Calling `AuthGuard('jwt')` produces a guard class configured to run whichever Passport strategy was registered under the name `'jwt'`, which, per the previous note, is exactly `JwtStrategy`. `@UseGuards(AuthGuard('jwt'))` attaches that guard to this one route.

When a request hits `GET /auth/profile`, this guard runs before `getProfile()` ever executes. It triggers `JwtStrategy`'s full flow: extract the bearer token from the `Authorization` header, verify its signature against `MY_JWT_SECRET`, check it has not expired, decode its payload, and pass that payload into `validate()`, whose return value, `{ userId, email }`, becomes `req.user`. Only if all of that succeeds does `getProfile()` actually run. If the token is missing, malformed, expired, or signed with the wrong secret, the guard itself rejects the request with a `401 Unauthorized`, and `getProfile()`'s body never executes at all.

`@Request() req` is NestJS's parameter decorator for injecting the raw underlying request object (the same kind of object Express normally hands a route handler) directly into the method, and `req.user` is precisely the value the guard attached moments earlier. `getProfile()` does nothing beyond returning that value directly, so a successful call to this route responds with exactly `{"userId": "<the user's MongoDB _id>", "email": "<their email>"}`, nothing more, since that is the entire shape `validate()` constructs. This is a clean, minimal illustration of the whole point of the strategy and guard system: by the time your own route handler code runs, all the token extraction, signature verification, and expiry checking is already done, and you are simply left holding a trustworthy, already validated `req.user`.
