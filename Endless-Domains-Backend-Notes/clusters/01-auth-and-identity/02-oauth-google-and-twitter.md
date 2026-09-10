# 02. OAuth: Google (Live) and Twitter (Not Wired In)

## Google, a genuine second front door into the same user table

`GoogleAuthController` exposes exactly one route, `POST /google-auth`, taking a `TokenVerificationDto` (`{ token, source }`). The token here is a Google access token the frontend already obtained through Google's own client side sign in flow, this backend's job is only to verify that token actually belongs to a real Google account and then either log that person in or create their account.

```ts
// src/components/google-auth/google-auth.service.ts
constructor(private readonly configService: ConfigService, ...) {
    const clientID = this.configService.get(GoogleAuthEnum.GOOGLE_AUTH_CLIENT_ID);
    const clientSecret = this.configService.get(GoogleAuthEnum.GOOGLE_AUTH_CLIENT_SECRET);
    this.oauthClient = new google.auth.OAuth2(clientID, clientSecret);
}

async authenticate(tokenVerificationDto: TokenVerificationDto): Promise<ReturnLoginDto> {
    const tokenData = await this.oauthClient.getTokenInfo(tokenVerificationDto.token);
    const email = tokenData.email;
    const name = (await this.getUserData(tokenVerificationDto.token)).name;

    let existingUser = await this.userService.findByEmailWithoutRestrictions(email);
    if (existingUser) {
        if (existingUser.isDeleted) { throw new UnauthorizedException('Your account has been deleted.'); }
        if (existingUser.isBlocked) { throw new UnauthorizedException('Your account has been blocked.'); }
        await this.userService.updateTheUserSource(existingUser.id, tokenVerificationDto.source);
        const response = await this.handleResponse(existingUser.id);
        response.user = existingUser;
        return response;
    } else {
        const newUser = await this.registerUser(email, name);
        const response = await this.handleResponse(newUser.id);
        response.user = newUser;
        return response;
    }
}
```

`this.oauthClient.getTokenInfo(token)` is a real call out to Google's own token introspection endpoint (through the `googleapis` package), this is genuine, network verified proof that the token is valid and actually belongs to the email address returned, unlike a purely local signature check. `getUserData`, defined a little further down, makes a second Google API call (`google.oauth2('v2').userinfo.get`) specifically to pull the account's display name, since `getTokenInfo` alone does not return one. If the email is new, `registerUser` calls `this.userService.createWithGoogle(email, name)` to create an account with no password at all, meaning that account can only ever be logged into again through this same Google flow, or through the password reset flow if it ever needed to add a password later. If the email already exists, the account is reused directly rather than creating a duplicate, meaning a person who first signed up with `/auth/register` and a password, and later hits "sign in with Google" using the same email address, ends up back in the exact same account, with Google now able to authenticate into it as well.

Notice what is deliberately absent here compared to the plain email and password login in `AuthService.login`: there is no `isEmailVerified` check on `existingUser` before calling `handleResponse`. That is a defensible design choice rather than an oversight, since Google has already independently verified this person's control of that email address by issuing them a valid access token in the first place, asking them to additionally click a verification link mailed by this backend would be redundant. It is still worth naming as a real, deliberate asymmetry between two of this backend's five login paths, since it means `isEmailVerified` on the `User` entity does not mean the same thing consistently across every way a user can reach that record, for a password based account it means "clicked our email link," for a Google based account it may simply never get checked at all on that login path.

The comment directly inside the live method, `// First, check if user exists by email directly (no auth checks)`, next to a commented out block above it containing an entire earlier version of this same method (one that called `this.userService.getByEmail` and relied on catching an `UnauthorizedException` from that call to detect a brand new user, rather than the current, more direct `findByEmailWithoutRestrictions`), is worth reading side by side, it is a clear, in file record of this exact code being revised at least once, moving from an exception driven "new user" detection to an explicit existence check.

`handleResponse`, shared in shape with the equivalent private method in `Web3AuthService`, issues tokens through `AuthServiceInterface.getTokens` (the very same method covered in the previous note, meaning Google logged in users get access and refresh tokens signed with the exact same two secrets as password logins), persists the hashed refresh token, and emits its own domain-refresh background event, `googleAuthUserLogin.UpdateDomainDetailTable`, handled by `GoogleAuthService.updateDomainDetialTable` itself rather than by `AuthService`, following the same pattern documented in the previous note.

## Twitter, present in the codebase but not part of the running application

`TwitterAuthModule`, `TwitterAuthController`, and `TwitterAuthService` all exist, fully written, under `src/components/twitter-auth`, but a direct search of `app.module.ts` and every other module in this codebase for `TwitterAuthModule` turns up nothing, it is imported nowhere. That single fact matters more than anything else about this component, no route under `/auth/twitter` or `/twitter` currently exists on a running instance of this backend, this module is dead code from the application's point of view, however complete it otherwise looks.

That matters because of exactly what is sitting inside `twitter-auth-service.ts`:

```ts
// src/components/twitter-auth/twitter-auth-service.ts
async getRequestToken(): Promise<TwitterRequestTokenResponse> {
    return new Promise((resolve, reject) => {
        request.post({
            url: 'https://api.twitter.com/oauth/request_token',
            oauth: {
                oauth_callback: TwitterAuthEnum.CALLBACK_URL,
                consumer_key: 'nyQ6iNVFJHHrixmRCXwgiViIq',
                consumer_secret: 'Scuv26MuZ9jUkjyeWeDQ8jFiEAtyBUsYReZMOp5sXxQUfMTS1c',
            }
        }, (err, r, body) => { ... });
    });
}
```

A real looking Twitter OAuth 1.0a consumer key and consumer secret are hardcoded directly into this source file, in plain text, rather than read from `ConfigService` or `SecretsService` the way every other third party credential in this codebase is handled (see [03-configuration-and-secrets.md](../../03-configuration-and-secrets.md) for how that is supposed to work, and how strongly it recommends never doing this). `TwitterAuthEnum.CALLBACK_URL` is similarly hardcoded to `http://localhost:3000/twitter-callback`, a local development URL, which on its own is a strong signal that this integration was written and then abandoned mid development rather than ever finished and deployed. Whether this specific key pair is still an active, valid Twitter developer app credential is not something that can be verified by reading source code alone, but a hardcoded API secret sitting in version control is exactly the kind of thing worth raising with whoever manages this team's third party API credentials, regardless of whether the module using it currently runs, since the string itself is still sitting in this repository's git history either way.

Structurally, even if `TwitterAuthModule` were wired into `AppModule`, it is not clear the login flow would work as intended, `TwitterAuthController.authenticate` reads `req.user` expecting some upstream Passport strategy to have already populated it, but no Twitter `PassportStrategy` subclass exists anywhere in this folder, and no `@UseGuards()` decorator appears on any route in `TwitterAuthController`. The `getAccessToken` handler manually copies OAuth values onto `req.body` and calls Express's own `next()` to fall through to the next handler, which is an unusual, hand rolled way to chain two route handlers together and does not match the guard based pattern used by every other authentication flow in this codebase.
