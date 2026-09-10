# 01. Password Login and JWT Tokens

## The shape of the module

`src/components/auth/auth.module.ts` wires together `AuthController` and `AuthService` behind an `AuthServiceInterface` token, the same interface plus dependency injection token pattern used everywhere in this codebase. What is worth noticing here specifically is how much this one module pulls in beyond just user and JWT concerns, `MoralisModule`, `DomainDetailModule`, and four separate Alchemy modules (`ArbAlchemyModule`, `UdBaseAlchemyModule`, `EnsAlchemyModule`, `UdAlchemyModule`, `FreenameAlchemyModule`) are all imported directly into `AuthModule`. That is because `AuthService` itself listens for its own login event and kicks off a background job to refresh a logged in user's on chain domain holdings, covered at the end of this note, a genuine example of a module whose responsibilities have grown past what its name suggests.

## Registering a new user

```ts
// src/components/auth/auth.service.ts
async register(registerUserDto: RegisterUserDto): Promise<void> {
    const hashedPassword = await bcrypt.hash(registerUserDto.password, 10);
    const createUserDto = new CreateUserDto();
    createUserDto.email = registerUserDto.email.toLowerCase();
    createUserDto.password = hashedPassword;
    createUserDto.role = registerUserDto.role === 'affiliate' ? 'affiliate' : 'user';
    try {
        const userSaved = await this.userService.create(createUserDto);
        const tokens = await this.getTokens(userSaved.id);
        await this.userService.setCurrentRefreshToken(tokens.refreshToken, userSaved.id);
        ...
        await this.emailVerificationService.sendVerificationEmail(userSaved.email, "user");
    } catch (error: any) {
        if (error?.code === PostgresErrorCode.UNIQUE_VIOLATION) {
            throw new BadRequestException(UserErrorMessage.USER_WITH_THIS_EMAIL_ALREADY_EXISTS);
        }
        throw new InternalServerErrorException(CommonErrorMessage.SOMETHING_WENT_WRONG);
    }
}
```

Registration hashes the password with `bcrypt.hash(..., 10)`, ten being the salt round count, saves the user, issues both an access and a refresh token immediately (so a freshly registered user is already logged in before they verify their email), and fires off a verification email through `EmailVerificationServiceInterface`, covered in its own note. Two things worth noticing precisely: the role passed in from the client is only ever allowed to become `'affiliate'` or `'user'`, `registerUserDto.role === 'affiliate' ? 'affiliate' : 'user'` means any other string a client sends is silently normalized down to `'user'`, so a client cannot self assign an admin style role through this endpoint. And the duplicate email check works by catching a real Postgres unique constraint violation error code rather than doing a pre check query first, a save first, ask forgiveness later pattern that avoids a race between two simultaneous registrations for the same email.

`RegisterUserDto` only requires `@MinLength(8)` on the password field, it does not apply the shared `PASSWORD_REGEX` pattern used elsewhere in this same file's sibling DTOs. Concretely, that means `register` and `login` (`LogInDto` has the identical `@MinLength(8)` only) will happily accept a password like `"aaaaaaaa"`, eight lowercase letters and nothing else, while `ChangePasswordDto.newPassword` and `ResetPasswordDto.password`, both defined a few files over in the same `dto` folder, additionally require `@Matches(PASSWORD_REGEX, { message: PASSWORD_VALIDATION_MESSAGE })`, meaning a password must include an uppercase letter, a lowercase letter, a digit, and a special character. This is a real, verifiable inconsistency between two entry points into the exact same password field on the exact same user, not a hypothetical one, the two DTOs sit right next to each other in `src/components/auth/dto/`.

## Logging in

```ts
async login(logInDto: LogInDto): Promise<ReturnLoginDto> {
    const normalizedEmail = normalizeEmail(logInDto.email);
    const user = await this.userService.getByEmail(normalizedEmail);
    if (!user) { throw new UnauthorizedException(UserErrorMessage.INVALID_ATTEMPT); }
    if (user.isDeleted) { throw new UnauthorizedException('Your account has been deleted...'); }
    if (user.isBlocked) { throw new UnauthorizedException('Your account has been blocked...'); }
    const passwordFromDB = await this.userService.getPasswordByUserId(user.id);
    if (!passwordFromDB) { throw new ForbiddenException(UserErrorMessage.INVALID_ATTEMPT); }
    if (!user.isEmailVerified) { throw new BadRequestException(UserErrorMessage.EMAIL_NOT_VERIFIED_YET); }
    await this.userService.updateTheUserSource(user.id, logInDto.source);
    await this.verifyPassword(logInDto.password, passwordFromDB);
    const tokens = await this.getTokens(user.id);
    await this.userService.setCurrentRefreshToken(tokens.refreshToken, user.id);
    this.eventEmitter.emit(`userlogin.UpdateDomainDetailTable`, new UpdateDomainDetailTableAfterLoginEvent(user.id));
    this.eventEmitter.emit('reputation.login', { userId: user.id });
    return returnLoginDto;
}
```

Login runs through five gates in a specific order before it ever touches the password, missing user, deleted account, blocked account, missing password hash on the record, and unverified email, all of which throw before `verifyPassword` is even called. `verifyPassword` itself is a thin wrapper around `bcrypt.compare(plainTextPassword, hashedPassword)` that throws `UnauthorizedException` on a mismatch. Once the password checks out, `getTokens` issues a fresh access and refresh token pair, the new refresh token is persisted (hashed, see below) against the user, and two application events fire, one to refresh the user's on chain domain data in the background and one for a reputation scoring system, both of which run asynchronously and do not block the login response itself.

## Where the two JWT secrets actually come from

```ts
constructor(
    ... ,
    private readonly configService: ConfigService,
    private readonly secretsService: SecretsService
) {
    this.JWT_ACCESS_TOKEN_SECRET = this.configService.get<string>('JWT_ACCESS_TOKEN_SECRET');
    this.JWT_REFRESH_TOKEN_SECRET = this.configService.get<string>('JWT_REFRESH_TOKEN_SECRET');
    ...
}
```

Inside `AuthService` itself, both secrets are read the "correct" way for this codebase, through `ConfigService.get(...)`, which as covered in [03-configuration-and-secrets.md](../../03-configuration-and-secrets.md) is populated at boot from the AWS Secrets Manager bundle named by `AWS_MANAGER`, not from a local `.env` file. `getTokens` then signs both tokens with `jwtService.signAsync`, one with `JWT_ACCESS_TOKEN_SECRET` and `JWT_ACCESS_TOKEN_EXPIRATION`, one with `JWT_REFRESH_TOKEN_SECRET` and `JWT_REFRESH_TOKEN_EXPIRATION`, both durations themselves also pulled from `ConfigService`. The payload signed into both tokens is intentionally minimal, `{ userId: userId }`, nothing else, meaning a route handler that needs the user's role or other attributes has to look the user up again rather than trusting anything embedded in the token.

Where this gets inconsistent is in how the token is later verified, not signed. `access-token.strategy.ts`, the Passport strategy `AccessTokenGuard` actually delegates to, reads the same secret a different way:

```ts
// src/components/auth/strategies/access-token.strategy.ts
constructor() {
    super({
        jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
        secretOrKey: process.env.JWT_ACCESS_TOKEN_SECRET
    });
}
```

This reads `process.env.JWT_ACCESS_TOKEN_SECRET` directly, at class construction time, bypassing `ConfigService` and `SecretsService` entirely. Compare that to `refresh-token.strategy.ts` sitting right next to it in the same folder:

```ts
// src/components/auth/strategies/refresh-token.strategy.ts
constructor(private readonly secretsService: SecretsService) {
    super({
        jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
        secretOrKeyProvider: async () => this.JWT_REFRESH_TOKEN_SECRET,
        passReqToCallback: true
    });
}
async onModuleInit() {
    const secrets = await this.secretsService.getSecret(process.env.AWS_MANAGER);
    this.JWT_REFRESH_TOKEN_SECRET = secrets.JWT_REFRESH_TOKEN_SECRET;
}
```

The refresh token strategy correctly fetches its secret from `SecretsService` asynchronously in `onModuleInit`, exactly the pattern the rest of this codebase follows for real configuration. The access token strategy, the one guarding almost every authenticated route in the app through `AccessTokenGuard`, does not, it reads a plain OS level environment variable. For this to work correctly in production, `JWT_ACCESS_TOKEN_SECRET` has to be set as an actual environment variable on the running container in addition to living inside the AWS secret bundle, since nothing in `main.ts` or `ConfigModule.forRoot` copies AWS Secrets Manager values onto `process.env`. Two guards elsewhere in this same cluster, `SuperAdminAccessGuard` and `MarketingAccessGuard` (covered in [05-rbac-and-guards.md](05-rbac-and-guards.md)), read the same secret a third way, through `ConfigService.get('JWT_ACCESS_TOKEN_SECRET')` directly inside the guard rather than through a Passport strategy at all. Three different code paths reading what is supposed to be the exact same secret value, three different ways, is worth being aware of if that secret is ever rotated, since missing even one of those paths during a rotation would quietly break authentication for whichever guard still holds the old value cached from `process.env` or a stale `ConfigService` snapshot.

## Refreshing tokens, and the two different hashing algorithms in play

```ts
async refreshTokens(userId: string, refreshToken: string): Promise<TokenResponseDto> {
    const refreshTokenFromDB = await this.userService.getHashedRefreshTokenByUserId(userId);
    if (!refreshTokenFromDB) throw new UnauthorizedException('Access Denied');
    const refreshTokenMatches = await verifyHash(refreshTokenFromDB, refreshToken);
    if (!refreshTokenMatches) throw new UnauthorizedException('Access Denied');
    const tokens = await this.getTokens(userId);
    await this.userService.setCurrentRefreshToken(tokens.refreshToken, userId);
    return new TokenResponseDto(tokens.accessToken, tokens.refreshToken);
}
```

`GET /auth/refresh-token` is protected by `RefreshTokenGuard`, which is just `AuthGuard('jwt-refresh')`, meaning the refresh token itself has to arrive as a bearer token in the `Authorization` header and pass the `RefreshTokenStrategy`'s `validate()` before this method is even called, `validate()` there reads the raw token back off the request (`req.get('Authorization').replace('Bearer', '').trim()`) and attaches it to `req.user.refreshToken` so the controller can hand it to `refreshTokens` alongside the userId decoded from the token's own payload.

The refresh token itself is never stored in the database in plaintext. `setCurrentRefreshToken`, over in `user.service.ts`, hashes it before saving:

```ts
// src/components/user/user.service.ts
async setCurrentRefreshToken(refreshToken: string, id: string): Promise<void> {
    const hashedRefreshToken = await hashData(refreshToken);
    await this.userRepo.update(id, { currentHashedRefreshToken: hashedRefreshToken, lastActiveAt: new Date() });
}
```

and `hashData`/`verifyHash`, defined in `src/@core/utils/helper.ts`, are thin wrappers around `argon2.hash` and `argon2.verify`, not `bcrypt`:

```ts
// src/@core/utils/helper.ts
export function hashData(data: string): Promise<string> {
    return argon2.hash(data);
}
export function verifyHash(hashtoVerify: string, data: string): Promise<boolean> {
    return argon2.verify(hashtoVerify, data);
}
```

So this one login flow genuinely uses two different, unrelated password hashing algorithms for two different secrets on the same user record, `bcrypt` for the login password itself (in `AuthService.register`, `AuthService.login`, `AuthService.changePassword`, and `AuthService.resetPassword`, all calling `bcrypt.hash`/`bcrypt.compare` directly), and `argon2` for the refresh token (through `hashData`/`verifyHash` in `user.service.ts`). Both are legitimate, currently recommended password hashing algorithms, this is not a security problem on its own, but it is a detail worth knowing before you go looking for "the" hashing function in this codebase and assume there is only one.

Refresh tokens in this codebase are always rotated on use, `refreshTokens` calls `getTokens` again and immediately overwrites the stored hash with `setCurrentRefreshToken`, so a given refresh token can only successfully be exchanged once before a new one replaces it. Note also, from the shipped `.env.sample`, that `JWT_REFRESH_TOKEN_EXPIRATION=20s` is the sample value, twenty seconds, which is almost certainly a placeholder rather than the real value configured in the production AWS secret bundle, but it is worth flagging that if that literal value were ever copied into a real environment as is, refresh tokens would expire faster than most user sessions would need them.

## Logout and the cookie connection to `main.ts`

```ts
async logout(userId: string, res: any): Promise<void> {
    await this.userService.removeRefreshToken(userId);
}
```

`GET /auth/logout`, behind `AccessTokenGuard`, simply nulls out `currentHashedRefreshToken` on the user's record through `removeRefreshToken`, which means any refresh token issued before that moment stops matching the hash check in `refreshTokens` and can never be exchanged again, this is a real, server enforced logout rather than a purely client side "forget the token" gesture.

There is a second, separate logout path, `GET /auth/logout-cookies`, that exists specifically because some part of this system issues the access and refresh tokens as HttpOnly cookies rather than (or in addition to) returning them in the JSON response body, tying directly back to `app.use(cookieParser())` in `main.ts` covered in the architecture note:

```ts
async logouCookies(res: ExpressResponse): Promise<void> {
    const secrets = await this.secretsService.getSecret(process.env.AWS_MANAGER);
    res.clearCookie(secrets.ACCESS_TOKEN, {
        httpOnly: true, secure: true, sameSite: 'none', domain: '.endlessdomains.io',
    });
    res.clearCookie(secrets.REFRESH_TOKEN, {
        httpOnly: true, secure: true, sameSite: 'none', domain: '.endlessdomains.io',
    });
}
```

Notice that the cookie *names* themselves, `secrets.ACCESS_TOKEN` and `secrets.REFRESH_TOKEN`, come out of the AWS secret bundle rather than being literal strings like `"access_token"`, and that `domain: '.endlessdomains.io'` with `sameSite: 'none'` is exactly the cross subdomain, cross origin cookie configuration that the `credentials: true` CORS setting in `main.ts` exists to support. The commented out block directly beneath each `clearCookie` call, an alternate `httpOnly: true, secure: false, sameSite: 'lax', domain: 'localhost'` configuration, is a leftover local development version of the same call, left in place as a comment rather than removed, a small but genuine trace of how this endpoint gets tested locally versus how it actually runs in production.

## Forgot password, reset password, and change password

`forgotPassword` signs a short lived JWT (`JWT_VERIFICATION_TOKEN_SECRET`, a third distinct secret from the access and refresh ones) containing only `{ email }`, stores that exact token string on the user's `forgot_password_verification` column, and emails a link containing it. `resetPassword` decodes that token, and specifically checks that the token passed back in from the reset password form still matches what is stored on the user record (`user.forgot_password_verification == token`), then immediately overwrites that column with a throwaway string (`` `${user.email}-${user.id}-${new Date().getDate().toString()}` ``) so the same reset link cannot be used a second time even though the JWT signature itself would still verify until it expires. That column level check is what actually makes the reset link single use, the JWT's own expiry alone would not be enough since a JWT is stateless and would otherwise remain valid for its full expiration window no matter how many times it is presented. `changePassword`, by contrast, requires no such token, it is a `PUT /auth/change-password` route behind `AccessTokenGuard`, so the caller already has a valid access token, and the service only re-checks the caller's *current* password against the stored bcrypt hash before allowing the new one through.

## The background job login quietly triggers

Both `login` and `register` (and, as covered in later notes, the Google and Web3 login paths) end by emitting an event that this same `AuthService` also listens for on itself:

```ts
@OnEvent('userlogin.UpdateDomainDetailTable', { async: true })
async updateDomainDetialTable(payload: UpdateDomainDetailTableAfterLoginEvent): Promise<void> {
    const response = await this.userService.getByIdWithAllWalletAddress(payload.userId);
    ...
    if (evmWallets && evmWallets.walletAddress !== null) {
        const UDAlchmeyData = await this.udAlchemyService.getAllNFTsOwnedbyAddress(...);
        ...
    }
    if (solanaWallets) {
        const SolanaData = await this.MoralisService.nftsOwnedbySolanaWalletAddress(...);
        ...
    }
    await this.DomainDetailService.bulkUpdateDomainDetailBlockChain(data, payload.userId);
}
```

This is why `AuthModule` imports so many blockchain data provider modules despite being, on paper, an authentication module. Every successful login kicks off a background refresh of whatever domains the logging in user's connected wallets actually hold on chain, across five different providers (Unstoppable Domains, ENS, Arbitrum, Binance Smart Chain, and a Base variant of Unstoppable Domains, plus Freename), so that the rest of the application always has reasonably fresh data without the user having to manually refresh anything. It runs asynchronously (`{ async: true }`), so a slow or failing call to any of these five providers never delays or fails the login response itself, it only affects how fresh the user's domain listing looks afterward.
