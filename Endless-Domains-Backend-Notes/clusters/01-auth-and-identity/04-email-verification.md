# 04. Email Verification

## Why this is its own module rather than a method on `AuthService`

`EmailVerificationModule` is a small, standalone module exporting `EmailVerificationServiceInterface`, and it is imported into `AuthModule`, `Web3AuthModule` (indirectly, through `AuthModule`), and several other modules outside this cluster entirely, since the same service also sends a wide range of unrelated transactional emails, low balance warnings, domain expiry reminders, affiliate reports, and more, all through the same `EventEmitter2` based pattern used for password reset emails in the previous notes. Only `sendVerificationEmail`, `sendVerificationEmailPostLogin`, `verifyEmail`, `resendVerificationEmail`, and `verifyEmailAfterLogin` are actually about identity, the rest of the interface is email infrastructure that happens to live in the same file.

## Its own JWT secret, separate from access and refresh

```ts
// src/components/email-verification/email-verification.service.ts
this.JWT_VERIFICATION_TOKEN_SECRET = this.configService.get<string>('JWT_VERIFICATION_TOKEN_SECRET');
this.JWT_VERIFICATION_TOKEN_EXPIRATION_TIME = this.configService.get<string>('JWT_VERIFICATION_TOKEN_EXPIRATION_TIME');
```

This is a third distinct JWT secret in this cluster, separate from `JWT_ACCESS_TOKEN_SECRET` and `JWT_REFRESH_TOKEN_SECRET`, and it is reused for two unrelated purposes across this cluster, email verification tokens here, and the forgot password tokens signed in `AuthService.forgotPassword` and `RbacUserRepo.forgotPassword` (both covered elsewhere in this cluster). Sharing one secret across two different token *purposes* is a real design choice worth naming, since nothing in the payload of either token type distinguishes "this is a verification token" from "this is a password reset token" beyond the fields present, `{ email }` in both cases, meaning a verification token minted for one purpose would, if ever pointed at the other endpoint, decode and pass signature verification cleanly, only the different expiration windows configured on each call site meaningfully separate them in practice.

## Sending the link

```ts
async sendVerificationEmail(email: string, isAffiliateUser?: string): Promise<void> {
    const payload: VerificationTokenPayload = { email };
    const token = this.jwtService.sign(payload, {
        secret: this.JWT_VERIFICATION_TOKEN_SECRET,
        expiresIn: this.JWT_VERIFICATION_TOKEN_EXPIRATION_TIME
    });
    await this.userService.updateSentEmailCount(email);
    const url = `${this.BACKEND_API_URL}/email-verification/verify?token=${token}`;
    let mailType = isAffiliateUser === 'affiliate' ? VERIFY_EMAIL_AFFILIATE : VERIFY_EMAIL;
    this.eventEmitter.emit(mailType, new VerifyEmailDto({ to: email, ..., partialContext: new VerifyEmailBodyContextDto({ ..., emailVerificationLink: url, ... }) }));
}
```

The token is a plain, stateless JWT with only `{ email }` in its payload, no random component, no reference to a stored value anywhere on the user record the way the forgot password flow's `forgot_password_verification` column provides. That means this specific link is not single use in the way a password reset link is, verifying the same email twice with the same still valid token would simply be blocked by the separate `if (user.isEmailVerified) { throw new BadRequestException(...) }` check in `verifyEmail`, described below, rather than by the token itself being consumed, the token stays valid and would decode successfully as many times as it is presented until it expires or the account gets verified through some other path. In practice this is a difference without a real consequence, since the only thing a valid token lets you do is mark an email verified, and doing that twice is harmless and already blocked, but it is a structurally different mechanism from the password reset token next to it in this same cluster, worth knowing so you do not assume both work the same way.

The link itself points at `BACKEND_API_URL`, not `WEB_CLIENT_URL`, meaning the user clicks through directly into this backend first, not into the frontend, which is what makes the `@Redirect(...)` decorator on the controller below necessary.

## Consuming the link: two routes for two contexts

```ts
// src/components/email-verification/email-verification.controller.ts
@Get('verify')
@Redirect('https://endlessdomains.io')
public async verifyEmail(@Query('token') token: string): Promise<EmailVerificationResponse> {
    await this.emailVerificationService.verifyEmail(token);
    return { url: this.WEB_CLIENT_URL + '/login?verification=success' };
}

@Get('verify-email')
@Redirect('https://endlessdomains.io')
public async verifyEmailAfterLogin(@Query('token') token: string): Promise<EmailVerificationResponse> {
    await this.emailVerificationService.verifyEmailAfterLogin(token);
    return { url: this.WEB_CLIENT_URL + '/profile/user' };
}
```

Both routes use Nest's `@Redirect()` decorator with a hardcoded fallback URL, and return an object shaped `{ url }` that Nest uses to override that fallback with the real destination once the handler runs, this is the standard Nest pattern for a dynamic redirect target, the string passed to the decorator itself is only ever seen if the handler throws before reaching its `return`. `GET /email-verification/verify` is the one linked from the registration email, and sends a successfully verified user to a `/login?verification=success` page. `GET /email-verification/verify-email` is a second, separate link sent by `sendVerificationEmailPostLogin` (using its own third expiration constant, `JWT_VERIFICATION_TOKEN_EXPIRATION_TIME_FOR_POST_VERIFY`), used for a user who registered, is already logged in, but still has not verified their email, and sends them to `/profile/user` instead of the login page, since they are already logged in and do not need to log in again.

`verifyEmail` and `verifyEmailAfterLogin` share almost identical bodies, decode the token, look the user up by the email inside it, reject with `EmailVerificationErrorMessage.EMAIL_ALREADY_VERIFIED` if `isEmailVerified` is already `true`, then call `userService.markEmailAsVerified(email)` and emit a welcome email event, `verifyEmail` additionally branches its welcome email template based on `user.role === 'affiliate'`, sending `WELCOME_AFFILIATE` instead of the plain `WELCOME` mail type, while `verifyEmailAfterLogin` always sends the plain one regardless of role, a small, easily missed asymmetry between two otherwise near identical methods.

## Decoding and its failure modes

```ts
private async decodeVerificationToken(token: string): Promise<string> {
    try {
        const payload = await this.jwtService.verify(token, { secret: this.JWT_VERIFICATION_TOKEN_SECRET });
        if (typeof payload === 'object' && 'email' in payload) {
            return payload.email;
        }
        throw new BadRequestException(EmailVerificationErrorMessage.EMAIL_NOT_FOUND);
    } catch (error) {
        if (error?.name === 'TokenExpiredError') {
            throw new BadRequestException(EmailVerificationErrorMessage.EMAIL_VERIFICATION_TOKE_EXPIRED);
        }
        throw new BadRequestException(EmailVerificationErrorMessage.INVALID_EMAIL_VERIFICATION_TOKEN);
    }
}
```

This distinguishes an expired token from every other kind of invalid token by checking `error.name === 'TokenExpiredError'`, a detail specific to how `jsonwebtoken` (which `@nestjs/jwt` wraps) reports its own errors, and surfaces a distinctly worded message for each case, so a frontend can tell a user "this link expired, request a new one" instead of a generic "invalid link" for every failure. The exact same pattern, down to the same `TokenExpiredError` check, appears again in `AuthService.decodeVerificationToken` for the password reset flow and a third time in `RbacUserRepo`'s equivalent method, three separate, near identical implementations of the same JWT decoding logic living in three different files rather than one shared helper, worth knowing if you ever need to change how any one of them behaves.
