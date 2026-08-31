# 05. The Passport JWT Strategy and Protected Routes

## What Passport and a "strategy" actually are, briefly

Passport is a widely used Node.js authentication middleware library, and it works around the concept of a "strategy," a small, pluggable unit that knows how to authenticate a request one specific way, a username and password strategy, a Google login strategy, a bearer token strategy, and so on. `@nestjs/passport` is NestJS's own thin integration layer on top of Passport, letting you define a strategy as an ordinary injectable NestJS class and then activate it on any route using the standard `@UseGuards()` mechanism, exactly like any other guard. This project defines exactly one strategy, for verifying JWTs, using the `passport-jwt` package's `Strategy` class as its foundation.

## `jwt.strategy.ts` in full

```ts
import { PassportStrategy } from "@nestjs/passport";
import { ExtractJwt, Strategy } from 'passport-jwt';
import { Injectable } from "@nestjs/common";
import { ConfigService } from "@nestjs/config";

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy){
    constructor(configService: ConfigService){
        super({
            jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
            secretOrKey: configService.get<string>('MY_JWT_SECRET')
        });
    }

    async validate(payload:any) {
        return { userId: payload.sub, email: payload.email }
    }
}
```

## `extends PassportStrategy(Strategy)`

`PassportStrategy` is a function from `@nestjs/passport` that takes a raw Passport strategy class, here `Strategy` imported from `passport-jwt`, and returns a new class wrapped so it fits NestJS's own dependency injection and guard system. `export class JwtStrategy extends PassportStrategy(Strategy)` is a class extending the result of calling that function, this is a fairly advanced looking piece of syntax the first time you see it (a class expression being extended directly, rather than a named class), but the effect is straightforward: `JwtStrategy` becomes both a normal, injectable NestJS provider (`@Injectable()` on top confirms that) and a fully functional Passport strategy at the same time.

By convention, when `PassportStrategy` wraps a strategy without an explicit name being passed as a second argument, the resulting strategy gets registered under the strategy's own default Passport name, and for `passport-jwt`'s `Strategy` class that default name is `'jwt'`. That name is exactly what shows up later in `AuthGuard('jwt')` on the controller, this is the thread connecting the two files, they are not coupled by anything explicit in this code, they are coupled by both independently agreeing on the string `'jwt'`.

## The constructor and `super({...})`

```ts
constructor(configService: ConfigService){
    super({
        jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
        secretOrKey: configService.get<string>('MY_JWT_SECRET')
    });
}
```

`ConfigService` is injected into `JwtStrategy`'s constructor the ordinary way, and immediately used to configure the underlying `passport-jwt` strategy through the `super({...})` call, which passes an options object up to `passport-jwt`'s own `Strategy` constructor.

`jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken()` tells `passport-jwt` exactly where to look for the token on an incoming request. `ExtractJwt` is a small helper object `passport-jwt` provides with several ready made extraction strategies, and `.fromAuthHeaderAsBearerToken()` is the standard one: it looks for an `Authorization` header formatted as `Bearer <token>` and pulls out just the `<token>` part. This is why a protected request in this project must be sent with a header that looks like `Authorization: Bearer eyJhbGciOi...`, that exact header shape is what this one line is configured to expect.

`secretOrKey: configService.get<string>('MY_JWT_SECRET')` is the second, and just as important, appearance of the same environment variable seen in the previous note, this time on the verifying side rather than the signing side. Since JWT signing and verifying with `passport-jwt`'s default HMAC based algorithm both use the exact same secret, this line has to read the identical `MY_JWT_SECRET` value that `JwtModule.registerAsync` used in `auth.module.ts`, and it does, both read it through `ConfigService` from the same environment variable name, so as long as that variable is set once in the environment, both sides of the token lifecycle, signing in `login()` and verifying here, agree on the same secret. Again, to be explicit and fair to this code: the secret is genuinely not hardcoded anywhere, it is supplied through configuration in both places it is needed.

This is also exactly the point where the earlier gap about a missing `MY_JWT_SECRET` becomes a real, loud failure rather than a quiet one. `passport-jwt`'s own `Strategy` constructor actively checks, at construction time, that it was given either a `secretOrKey` or a `secretOrKeyProvider`, and if both are missing, it throws immediately, before the app finishes starting up. Since `JwtStrategy` is listed in `AuthModule`'s `providers` array, Nest constructs it eagerly while assembling the dependency injection container during application bootstrap, not lazily on the first request. In practice, this means that if `MY_JWT_SECRET` is not set anywhere in the environment when this app starts, the entire application will fail to boot at all, with an error thrown from inside `passport-jwt`, rather than the app quietly starting up and only failing once someone actually tries to log in. That is, on balance, a reasonable failure mode, failing loudly at startup is safer than silently issuing tokens signed with `undefined`, but it is worth knowing exactly why the app refuses to start if that one environment variable is missing.

## `validate()`, and what it means for `req.user`

```ts
async validate(payload:any) {
    return { userId: payload.sub, email: payload.email }
}
```

Once `passport-jwt` has extracted a token from the `Authorization` header and successfully verified its signature against `secretOrKey` (and confirmed it has not expired, using the `exp` claim from the previous note), it decodes the token's payload and hands it to this `validate()` method. This is the one method every Passport strategy in NestJS is required to implement, and its return value is not just a return value in the ordinary sense, whatever `validate()` returns becomes the value NestJS attaches to `request.user` for the rest of that request's lifecycle.

Here, `validate()` receives the full decoded payload (recall from the previous note, this is `{ email, sub, iat, exp }`), and deliberately reshapes it before returning: `{ userId: payload.sub, email: payload.email }`. Notice the renaming, the token's `sub` claim becomes `userId` on `req.user`, while `email` keeps its name. This means any later code in this project that reads `req.user.userId` is reading the same MongoDB `ObjectId` that was embedded as `sub` when the token was signed back in `login()`. This reshaping step is a good habit in general, since it means the raw JWT payload shape and the application's internal "current user" shape are not forced to always be identical, even though in this small project the two happen to differ only in that one field name.

Note also what `validate()` does not do here: it does not query MongoDB again to confirm the user referenced by `payload.sub` still actually exists, and it does not check anything beyond what the signature verification already guarantees. This means a valid, unexpired token for a user who was later deleted from the database would still be accepted by this guard, `req.user` would simply contain a `userId` and `email` for an account that no longer exists. That is a common simplification in small JWT implementations (looking the user up again on every request adds a database round trip to every protected request), but it is worth knowing this project does not do that extra check.

## Where this guard actually gets used

There is exactly one place in this entire codebase where this strategy is activated, `GET /auth/profile` in `auth.controller.ts`, covered fully in the next note. `AppController`'s own single route has no guard at all and remains open to anyone. So the whole authentication mechanism built across this and the previous three notes exists to protect exactly one endpoint in this project, which is entirely appropriate for a small, focused example: the pattern demonstrated here, `@UseGuards(AuthGuard('jwt'))` combined with reading `req.user` afterward, is exactly how you would protect any number of additional routes in a larger application built on this same foundation.
