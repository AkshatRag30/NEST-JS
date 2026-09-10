# 05. RBAC, the Admin Login, and the Shared Guards

## What `user-rbac` actually is, checked against the real entity

The name suggests a full role and permission model, roles that map to a defined set of permissions, checked generically wherever needed. What is actually there, in `rbac-user.entity.ts`, is simpler than that:

```ts
// src/components/user-rbac/entity/rbac-user.entity.ts
@Entity({ name: 'tbl_role-base-user' })
export class RoleBaseUserEntity extends BaseEntity {
    public name: string;
    @Column({ unique: true }) public email: string;
    public walletAddress?: string;
    @Column({ nullable: false }) public password: string;
    public accessToken?: string;
    public twoFactorAuthenticationSecret?: string;
    @Column({ default: false }) public isTwoFactorAuthenticationEnabled: boolean;
    @Column({ default: false }) public isEmailVerified: boolean;
    public forgot_password_verification?: string;
    @Column({ nullable: false }) public role: string;
    @Column({ default: false }) public isDeleted: boolean;
    @Column({ type: 'text', array: true, nullable: true }) public pageAccess: string[];
}
```

`role` is a plain, unconstrained string column, not an enum, not a foreign key into a roles table, nothing at the database level stops it from being set to any string at all. There is no `Permission` entity and no join table anywhere in this folder mapping a role to a set of allowed actions. What this system actually uses instead is `pageAccess`, a raw Postgres text array column holding a literal list of page names an admin user is allowed to see, `example: ['user section', 'admin section']` from `CreateRbacUserDto`'s own Swagger annotation. In other words, this is closer to an access control list per user (a specific list of allowed pages, assigned individually to each admin when they are created or updated) than to a role based permission system where permissions are defined once per role and every user with that role automatically inherits them. `role` itself is used for exactly two things in the code actually read for this cluster, deciding which of three literal strings (`superAdmin`, `Admin`, `marketing`, defined in `AdminRole`) a guard will accept, and being displayed back to the admin UI, it is not looked up against any permissions table to decide what that role can do.

This is a meaningfully different, and simpler, shape than "real RBAC" in the textbook sense, and it is worth being precise about that difference rather than assuming the folder name guarantees a permission matrix exists just because it is called `user-rbac`.

```ts
// src/@core/common/enum/admin-role.enum.ts
// Role strings stored on RoleBaseUserEntity.role (a plain unconstrained string
// column). SuperAdminAccessGuard checks its two roles as inline literals and is
// left untouched; new guards (e.g. MarketingAccessGuard) should reference this
// enum instead of introducing more hardcoded literals.
export enum AdminRole {
    SUPER_ADMIN = 'superAdmin',
    ADMIN = 'Admin',
    MARKETING = 'marketing'
}
```

That comment is a direct, honest admission from whoever wrote `AdminRole`, that `SuperAdminAccessGuard` (covered below) still checks its two roles as inline string literals rather than through this enum, and that the enum itself was introduced specifically so the newer `MarketingAccessGuard` would not add a fourth hardcoded literal on top of the two already scattered through `SuperAdminAccessGuard`. It is a small, real example of a codebase incrementally cleaning up a pattern without going back to fix the original instance of it.

## Logging in as an admin: password, then TOTP

```ts
// src/components/user-rbac/rbac-user-repo.ts
async login(logInDto: RbacLogInDto): Promise<ReturnRbacLoginDto> {
    const user = await this.roleBaseUserRepo.findOne({ where: { email: logInDto.email } });
    if (!user) throw new UnauthorizedException('Invalid credentials');
    if (user.isDeleted) throw new UnauthorizedException('...deleted...');
    if (!user.isEmailVerified) throw new UnauthorizedException('Your email is not verified...');
    const isPasswordValid = await bcrypt.compare(logInDto.password, user.password);
    if (!isPasswordValid) throw new UnauthorizedException('Invalid credentials');

    if (user.twoFactorAuthenticationSecret) {
        return this.toOtpVerificationResponse(user);
    } else {
        return this.toFirstTimeLoginResponse(user);
    }
}
```

This is a real, two step, TOTP based (Time based One Time Password, the same standard behind Google Authenticator) two factor login, not just password plus a stored flag. The password check is a standard `bcrypt.compare`. What happens next branches on whether `twoFactorAuthenticationSecret` already exists on the row. The very first time a given admin logs in, it does not exist yet, so `toFirstTimeLoginResponse` runs:

```ts
private async toFirstTimeLoginResponse(user: RoleBaseUserEntity): Promise<ReturnRbacLoginDto> {
    const adminKey = this.configService.get<string>('ADMINKEY') || '';
    const prefix = adminKey.toUpperCase() === 'PROD' ? 'EDPROD' : 'EDSTAGE';
    const appLabel = `${prefix}(${user.email})`;
    const secret = speakeasy.generateSecret({ length: 20, name: appLabel, issuer: appLabel });
    user.twoFactorAuthenticationSecret = secret.base32;
    await this.roleBaseUserRepo.save(user);
    const qrCodeUrl = secret.otpauth_url;
    const otpSendURLPath = await qrcode.toDataURL(qrCodeUrl);
    ...
    returnLoginDto.qrCodeUrl = otpSendURLPath;
    returnLoginDto.secret = secret.base32;
    return returnLoginDto;
}
```

`speakeasy.generateSecret` mints a brand new random TOTP secret, saves it to the user's row immediately, and renders it as a scannable QR code (via the `qrcode` package, encoded as a data URL) so the admin can add this account to Google Authenticator or an equivalent app on the spot, this is the standard "first time 2FA setup" experience, and this backend generates the secret itself rather than trusting the client to supply one. On every subsequent login, `twoFactorAuthenticationSecret` already exists, so `toOtpVerificationResponse` runs instead, a much smaller response that just says "enter your OTP" without the QR code, and the real check happens on a second request:

```ts
async verifyOtp(rbac2FAloginotpdto: Rbac2FALogInOtpDto, email: string): Promise<Return2FAResponseDto> {
    const user = await this.roleBaseUserRepo.findOne({ where: { email } });
    if (!user) throw new UnauthorizedException('Invalid user');
    if (!user.twoFactorAuthenticationSecret) throw new UnauthorizedException('Session expired...');
    const isOtpValid = this.verifyTwoFactorCode(user.twoFactorAuthenticationSecret, rbac2FAloginotpdto);
    if (!isOtpValid) throw new UnauthorizedException('Invalid OTP');
    const token = this.generateJwt(user);
    return this.toReturnRbacLoginDto(user, token);
}

private verifyTwoFactorCode(secret: string, rbac2FAloginotpdto: Rbac2FALogInOtpDto): boolean {
    return speakeasy.totp.verify({ secret, encoding: 'base32', token: rbac2FAloginotpdto.token, window: 2 });
}
```

`POST /admin/rbacusers/verify-otp` has no `@UseGuards()` at all, it is a public route by design, since the admin is not holding any token yet at this point in the flow, only the OTP code from their authenticator app proves who they are here. `window: 2` allows the code from two TOTP intervals (roughly one minute) on either side of "now" to still count as valid, which is standard practice to tolerate small clock drift between server and phone, but it also means there is no explicit rate limiting or lockout on repeated OTP guesses on this specific route in the code read for this cluster, worth being aware of since a six digit TOTP code, brute forced without any throttling, is a meaningfully smaller search space than a real password.

## `generateJwt` reads the secret the same way `AccessTokenStrategy` does

```ts
private generateJwt(user: RoleBaseUserEntity): string {
    const payload = { userId: user.id, role: user.role, email: user.email };
    const secret_key = process.env.JWT_ACCESS_TOKEN_SECRET;
    return jwt.sign(payload, secret_key, { expiresIn: '24h' });
}
```

This admin JWT is signed directly with the `jsonwebtoken` package rather than Nest's `JwtService`, and it reads `process.env.JWT_ACCESS_TOKEN_SECRET` directly, the same pattern flagged in [01-password-login-and-jwt-tokens.md](01-password-login-and-jwt-tokens.md) for `AccessTokenStrategy`. It is signed with the exact same secret as an ordinary consumer's access token, meaning both token types are structurally interchangeable as far as the JWT signature itself is concerned, what actually separates an admin session from a consumer session is entirely which guard is checking the decoded payload afterward (whether it looks at `role`, covered next), not any difference in how the token itself was signed.

## The full guard lineup in `src/@core/common/guards`

`AccessTokenGuard` and `RefreshTokenGuard` are Passport based, both are one line subclasses of `AuthGuard('jwt')` and `AuthGuard('jwt-refresh')` respectively, delegating entirely to the two strategies covered in the first note of this cluster. Everything else in this folder implements `CanActivate` directly, decoding and checking a JWT by hand rather than going through Passport.

```ts
// src/@core/common/guards/super-admin-token.guard.ts
async canActivate(context: ExecutionContext): Promise<boolean> {
    const secretPrivateKey = this.configService.get<string>('JWT_ACCESS_TOKEN_SECRET');
    const authHeader = request.headers['authorization'];
    ...
    const decoded = jwt.verify(token, secretPrivateKey) as { role: string };
    request.user = decoded;
    if (decoded.role == this.ADMIN_ROLE) { return true; }
    if (decoded.role == 'Admin') { return true; }
    throw new UnauthorizedException('Access Denied: User does not have the required role');
}
```

`SuperAdminAccessGuard` is what protects almost every route in `RbacController`, it manually pulls the bearer token off the `Authorization` header, verifies it with `jsonwebtoken` directly against `JWT_ACCESS_TOKEN_SECRET` read through `ConfigService` (a third distinct way of sourcing this one secret, next to `process.env` in `AccessTokenStrategy` and `generateJwt` above), and then checks `decoded.role` against two hardcoded literal strings, `'superAdmin'` (via `this.ADMIN_ROLE`) and `'Admin'`, exactly the pair the comment on `AdminRole` above already admits were never migrated onto that enum. `MarketingAccessGuard` is a newer, separate guard built to the same shape on purpose, its own comment says it "mirrors `SuperAdminAccessGuard`'s JWT decode logic rather than modifying it, so none of that guard's existing call sites are affected," and it does reference `AdminRole.MARKETING` and `AdminRole.SUPER_ADMIN` rather than repeating string literals a third time. `AdminTokenGuard` is simpler still, and is not JWT based at all, it compares a raw `admin-token` request header directly against a single `ADMIN_TOKEN` value from `ConfigService`, a shared secret bearer token rather than a per user credential.

`IpRateLimitGuard` and `IpTokenRateLimitGuard` are both plain, in memory, per process rate limiters, keyed by IP alone or by `` `${ip}-${token}` `` respectively, and both carry their own doc comments explicitly admitting the same limitation, that under a multi worker deployment (this app documented elsewhere as running as several ECS tasks behind a load balancer) each worker keeps its own independent count, so the real effective limit becomes `maxRequests * worker_count` rather than the number actually configured, a known, accepted tradeoff for the specific low stakes routes they guard (forgot password, invoice download by token, a marketplace refresh endpoint) rather than an oversight. `AiDomainDiscoveryRateLimitGuard` and `AiDomainDiscoveryCircuitBreakerGuard`, in the same folder, solve a closely related problem for a different, unauthenticated AI feature using a signed identity cookie and a Postgres backed atomic counter instead of an in memory Map, specifically because that feature's guard comment explains the in memory approach was already known not to hold up across multiple ECS tasks, worth a glance if you want to see the more careful version of the same rate limiting problem solved once the team clearly cared about it holding up under real concurrent load.

## `RecaptchaGuard`: written, but neither correct nor used anywhere

```ts
// src/@core/guard/recaptcha.guard.ts
async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const secretKey = this.RE_CAPTCHA_SECRET;
    if (!request.headers['recaptchaValue']) return false;
    const verificationURL = 'https://www.google.com/recaptcha/api/siteverify?secret=' + secretKey + '&response=' + request.headers['recaptchaValue'];
    this.httpService.get(verificationURL).subscribe(() => {
        return true;
    });
}
```

Two separate problems sit in this one short method. First, `this.httpService.get(...)` returns an RxJS `Observable`, and `.subscribe(() => { return true; })` is fired without being awaited or returned, `canActivate` itself has no `return` statement after that call, so this method's actual return value, on every request that gets past the missing header check, is `undefined`. Nest treats a falsy `canActivate` result as "deny," so as written this guard would reject every request it is ever applied to, the `return true` inside the `subscribe` callback only sets the resolved value of an internal RxJS subscription that nothing is listening to, it does not make its way back out to Nest's guard machinery at all. Second, and this is what makes the first problem currently harmless rather than an active outage, a direct search of this entire `src` folder for `RecaptchaGuard` turns up exactly one match, the guard's own definition file, it is never referenced in a single `@UseGuards()` anywhere in the application, and `.env.sample` has no `RE_CAPTCHA_SECRET` entry either. This guard is fully unused today, a genuinely broken implementation sitting inert rather than a live bug, but worth fixing before anyone reaches for it expecting working recaptcha protection, since picking it up and adding it to a route as is would immediately and silently start rejecting every request to that route.

## Password validation, shared across every DTO that uses it

```ts
// src/@core/utils/password-validation.util.ts
export const PASSWORD_REGEX =
    /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[!@#$%^&*()_+\-=\[\]{};':"\\|,.<>/?]).{8,}$/;
export const PASSWORD_VALIDATION_MESSAGE =
    'Password must be at least 8 characters and include at least one uppercase letter, one lowercase letter, one number, and one special character.';
```

One shared regex, using lookaheads to independently require a lowercase letter, an uppercase letter, a digit, and a special character, plus a minimum length of eight, all in a single pattern. It is imported and applied, via `@Matches(PASSWORD_REGEX, ...)`, on `ChangePasswordDto.newPassword`, `ResetPasswordDto.password`, and `CreateRbacUserDto.password` (the admin creation DTO covered above). As already flagged in the first note of this cluster, it is conspicuously not applied on `RegisterUserDto.password`, `LogInDto.password`, or `RbacLogInDto.password`, all three of which only carry `@MinLength(8)`. Since a login DTO validating password strength would make no sense anyway (an existing password does not need to satisfy today's strength rules), the more meaningful gap is specifically `RegisterUserDto`, a brand new consumer account can be created today with a password like `"aaaaaaaa"`, while a brand new admin account created moments later through `CreateRbacUserDto` cannot.

## Returning the password hash in the response body

One more concrete, specific finding worth naming precisely. Compare the ordinary consumer facing `ReturnUserDto`:

```ts
// src/components/user/dto/return-user.dto.ts
export class ReturnUserDto {
    id: string; name: string; email: string; walletAddress?: string;
    isRegisteredWithGoogle: boolean; isEmailVerified: boolean; isBlocked: boolean; ...
    // no password field
}
```

against the three DTOs `user-rbac` actually returns to a caller:

```ts
// src/components/user-rbac/dto/return-rbacuser.dto.ts
export class ReturnRbacUserDto {
    id: string; name: string; email: string; password: string; accessToken: string; pageAccess: string[]; role: string;
}
// src/components/user-rbac/dto/return-rbac-login.dto.ts
export class ReturnRbacLoginDto {
    email: string; password: string; accessToken: string; qrCodeUrl: string; secret: string; message: string; role: string;
}
// src/components/user-rbac/dto/return-2fa-response-otp.dto.ts
export class Return2FAResponseDto {
    userId: string; email: string; password: string; accessToken: string; role: string; name: string; ...
}
```

All three RBAC response DTOs carry a `password` field, and the service methods that build them populate it directly from the entity, `returnUserDto.password = plainPassword` (the plaintext password, immediately after hashing it, in `createUser`) in `toReturnUserDto`, and `returnLoginDto.password = user.password` (the bcrypt hash straight from the database row) in `toReturnRbacLoginDto`. That means `POST /admin/rbacusers/register`, `POST /admin/rbacusers/login`, and `POST /admin/rbacusers/verify-otp` all include either the admin's plaintext password (only at creation time, since that is the one moment the plaintext value still exists in memory) or their bcrypt password hash directly in the JSON response body sent back over the wire. The consumer facing login and registration DTOs in the main `auth` and `user` components never do this, `ReturnUserDto` simply has no password field to populate. Whether or not this data ever gets rendered on screen by whatever admin frontend consumes this API, it is genuinely present in the HTTP response payload today, which means it will show up in browser dev tools, in any request logging or APM tooling that captures response bodies, and in any error reporting tool that snapshots a failed request, and that is worth raising as a real fix rather than filing away as low priority, since removing an already unnecessary field like this from a DTO is a small, low risk change.
