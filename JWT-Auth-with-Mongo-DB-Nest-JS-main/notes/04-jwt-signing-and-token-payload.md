# 04. JWT Signing and the Token Payload

## What a JWT actually is, briefly

A JSON Web Token is a compact string made of three parts separated by dots, a header, a payload, and a signature. The header and payload are just base64 encoded JSON, anyone can decode and read them without any secret at all, a JWT is not encrypted. What makes it trustworthy is the third part, the signature, a cryptographic value computed from the header, the payload, and a secret key known only to the server. When the server later receives a token back, it recomputes that signature using the same secret and checks it matches, if even one character of the header or payload was tampered with, the signature will not match, and the token is rejected. This is exactly the mechanism `@nestjs/jwt` and `passport-jwt` implement for this project, `JwtService.sign()` on the way out (this note), and the `JwtStrategy` verifying it on the way back in (the next note).

## Registering `JwtModule` in `auth.module.ts`

```ts
JwtModule.registerAsync({
  imports: [ConfigModule],
  inject: [ConfigService],
  useFactory: (config: ConfigService) => ({
    secret: config.get<string>('MY_JWT_SECRET'),
    signOptions: { expiresIn: '1h'}
  })
})
```

`JwtModule.registerAsync({...})` is the asynchronous form of registering `@nestjs/jwt`'s module, used instead of the simpler `JwtModule.register({...})` specifically because the configuration values it needs, the signing secret, are not known ahead of time as plain literals, they need to be read out of `ConfigService` once that service is available. `imports: [ConfigModule]` makes `ConfigService` available inside this registration's own dependency injection scope, and `inject: [ConfigService]` tells Nest to resolve a `ConfigService` instance and pass it as the argument to the function that follows. `useFactory: (config: ConfigService) => ({...})` is that function, it runs once, receives the resolved `ConfigService`, and returns the actual options object `JwtModule` will use internally to construct its `JwtService`.

Inside that returned object, `secret: config.get<string>('MY_JWT_SECRET')` is the single most important line in this whole note: this is where the signing secret actually comes from. It is read from an environment variable named `MY_JWT_SECRET` through `ConfigService`, the same `ConfigService` made globally available back in `app.module.ts`'s `ConfigModule.forRoot({ isGlobal: true })`. This is worth stating clearly and honestly: the secret is not hardcoded anywhere in this source code. There is no literal string like `'my-secret-key'` sitting in `auth.module.ts` or anywhere else in this repository, which is the correct approach, a JWT signing secret should always live outside source control, in an environment variable or a secrets manager, precisely so it cannot leak if the repository is ever made public or shared.

That said, there is a real, related gap worth flagging just as honestly: this project ships no `.env` file and no `.env.example` file (the `.gitignore` excludes `.env` and its local variants, and no example file exists anywhere in the repository), so there is no committed record of what `MY_JWT_SECRET` should even be named or look like, someone cloning this repository has to know to create that variable themselves with no template to follow. And, more seriously, nothing in this codebase validates that `MY_JWT_SECRET` is actually set before the app tries to use it. If it is missing, `config.get<string>('MY_JWT_SECRET')` simply returns `undefined`, and `secret: undefined` gets passed down into `JwtModule`'s setup. Whether that failure is silent or loud depends on which piece of code touches the secret first, and as covered in the next note, it is actually the Passport strategy, not the signing call, that fails first and loudest.

`signOptions: { expiresIn: '1h' }` sets the token's expiry. This is a standard `jsonwebtoken` option, `'1h'` is parsed as a duration string meaning one hour from the moment the token is signed. Practically, this means every access token issued by `login()` embeds an expiry timestamp (the standard `exp` claim) exactly one hour after it was created, and any request presenting that token after that hour has passed will be rejected by the Passport strategy as an expired token, forcing the user to log in again to get a fresh one. There is no refresh token mechanism anywhere in this project, once the hour is up, the only way back in is `POST /auth/login` again with the original email and password.

## The token payload: what actually goes inside

Back in `AuthService.login()`:

```ts
const payload = { email: user.email, sub: user._id };
return {
    access_token: this.jwtService.sign(payload),
}
```

The payload embedded in every token issued by this project is exactly two custom fields, `email` and `sub`. `email` is the user's email address, copied straight from the just fetched `User` document. `sub` is set to `user._id`, the user's MongoDB `ObjectId`. The name `sub` is not arbitrary, it is the standard JWT claim name for "subject," meaning "who is this token about," and using that standard name is a deliberate, idiomatic choice, even though nothing in this project's own code reads the claim by that specific standard meaning, it is simply good practice that other JWT tooling (and any future code added to this project) will recognize.

`this.jwtService.sign(payload)` takes that payload object, combines it with the `secret` and `signOptions` configured above, and produces the final signed token string. Beyond `email` and `sub`, `jsonwebtoken` (through `JwtService`) automatically adds a small number of standard claims on its own, most importantly `iat` (issued at, a timestamp) and `exp` (expiry, computed from `iat` plus the `expiresIn` option). Nothing else is embedded in this token, there is no role, no username, no list of permissions, just enough information to identify which user this token belongs to.

It is worth remembering, since a JWT's payload is only base64 encoded and never encrypted, that `email` and the raw MongoDB `_id` inside `sub` are both fully readable by anyone who has a copy of the token, without needing the secret at all, only forging a new, valid signature requires the secret. Neither piece of information here is especially sensitive on its own, but it is a useful habit to keep in mind for any JWT: never put anything genuinely secret, like a password or a payment detail, directly into the payload.
