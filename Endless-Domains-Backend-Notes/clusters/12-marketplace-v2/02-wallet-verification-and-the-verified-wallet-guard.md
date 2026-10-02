# 02. Wallet Verification and the Verified Wallet Guard

## The problem this solves

Marketplace v2 (see [01](01-what-marketplace-v2-is-and-the-module-map.md)) is built around wallet signatures. When a seller posts a signed Seaport order, the backend has to answer one question before it trusts anything else in that order: is the person holding this session really in control of the wallet that signed it? An order's `offerer` field is just an address in a JSON body, anybody can type any address there. A Seaport signature proves that the offerer's key signed the order, but it does not prove that the offerer is the same human as the logged in user making the HTTP request. Without a link between "this session" and "this wallet," a logged in user could submit an order someone else signed, or cancel and browse "my listings" for a wallet that is not theirs.

The original codebase already had wallet signatures, in `web3-auth` (covered in [the web3 wallet auth note](../01-auth-and-identity/03-web3-wallet-auth.md)), but they were used only for two things, logging in with a wallet, and attaching a wallet to an account. Once that happened, the access token that came out the other end was the same minimal `{ userId }` token as for a password login. Nothing in the session said "and by the way, this user proved control of wallet X at time T." So the B02 brief (branch `feat/t49-b02-jwt-wallet`, pull requests #943 and #944) introduced exactly that: two new optional JWT claims, `walletAddress` and `walletVerifiedAt`, a guard that reads them, and (in B02b) a dedicated endpoint pair for users who logged in with email and need to prove their wallet inside their existing session.

The nice thing about putting the proof inside the JWT is that it is stateless. Once the claim is signed into the token by the server, any route can check it without a database lookup, and nobody can forge it without the JWT secret. The tradeoff, which we will come back to, is that a claim inside a token cannot be revoked early, it lives until the token or the guard's freshness window says otherwise.

## The two new claims, and how they get into the token

```ts
// src/components/auth/interface/jwt-payload.interface.ts
export interface JwtPayload {
    userId: string;
    walletAddress?: string;
    walletVerifiedAt?: number;
}

export interface VerifiedWalletInfo {
    walletAddress: string;
    walletVerifiedAt: number;
}
```

`JwtPayload` gained two optional fields. `walletVerifiedAt` is a plain number, milliseconds since the Unix epoch (`Date.now()`), not seconds like the JWT's own `iat` and `exp` claims, keep that in mind if you ever decode the token on the frontend and compare. `VerifiedWalletInfo` is the same pair with both fields required, the type passed around internally when a caller has a proof to attach.

The single place tokens are minted, `AuthService.getTokens`, now accepts that optional proof:

```ts
// src/components/auth/auth.service.ts (line 276)
async getTokens(userId: string, verifiedWalletInfo?: VerifiedWalletInfo): Promise<TokenResponseDto> {
    const jwtPayload: JwtPayload = {
        userId: userId,
        ...(verifiedWalletInfo && {
            walletAddress: verifiedWalletInfo.walletAddress,
            walletVerifiedAt: verifiedWalletInfo.walletVerifiedAt
        })
    };

    const [accessToken, refreshToken] = await Promise.all([
        this.jwtService.signAsync(jwtPayload, {
            secret: this.JWT_ACCESS_TOKEN_SECRET,
            expiresIn: this.JWT_ACCESS_TOKEN_EXPIRATION
        }),
        this.jwtService.signAsync(jwtPayload, {
            secret: this.JWT_REFRESH_TOKEN_SECRET,
            expiresIn: this.JWT_REFRESH_TOKEN_EXPIRATION
        })
    ]);
    ...
}
```

The `...(verifiedWalletInfo && { ... })` trick is worth recognising because it shows up a lot in TypeScript: spreading `undefined` (or `false`) into an object literal adds nothing, so when no proof is passed, the payload is exactly `{ userId }` as before, byte for byte, and every existing caller (`register`, `login`, Google login and so on) keeps producing the same tokens they always did. Notice also that the same payload goes into both the access token and the refresh token, which matters for the refresh path below.

On the way back in, nothing new had to be written. `AccessTokenStrategy.validate(payload)` simply `return payload`, so whatever claims are in the access token appear on `req.user` after `AccessTokenGuard` runs. `RefreshTokenStrategy.validate(req, payload)` returns `{ ...payload, refreshToken }`, so the claims inside a refresh token appear on `req.user` after `RefreshTokenGuard` runs. To make that type safe, `RefreshTokenValidateResponseDto` gained two optional fields, `walletAddress?: string` and `walletVerifiedAt?: number` (its constructor is unchanged), and the Express type augmentation gained them too (covered further down).

## Three ways a session gets a verified wallet, and three ways it loses one

A token gets the claims in exactly three places today.

1. Logging in with a wallet. `POST /web3-auth/authenticate` now passes `{ walletAddress, walletVerifiedAt: Date.now() }` into `handleResponse`, which forwards it to `getTokens`. A wallet login is itself a fresh signature from that wallet, so it counts as proof.
2. Proving a wallet inside an existing session. `POST /marketplacev2/orders/auth/prove-wallet`, the B02b endpoint, for users who logged in with email (or Google) and have a wallet linked.
3. Refreshing. `GET /auth/refresh-token` now copies the claims from the refresh token being rotated into the new pair, without bumping the timestamp.

And a session ends up without them in three ways. Any password login, registration or Google login mints `{ userId }` only, by design. The legacy `POST /web3-auth/refresh-token` route calls `refreshTokens` without the new argument, so using it silently strips the claims (this is a bug, see the risks section). And of course an old token minted before this change has no claims at all.

## The web3 auth changes since baseline, line by line

```ts
// src/components/web3-auth/web3-auth.service.ts
async authenticate(web3AuthDto: Web3AuthDto): Promise<ReturnLoginDto> {
    const logger = new Logger(Web3AuthService.name + '-authenticate');
    web3AuthDto.walletAddress = ethers.getAddress(web3AuthDto.walletAddress);
    const nonce = await this.userService.getNonceByWalletAddress(web3AuthDto.walletAddress, web3AuthDto.network);

    const decodedAddress = this.decodeSignature(web3AuthDto.signature, nonce);
    if (web3AuthDto.walletAddress.toLowerCase() === decodedAddress.toLowerCase()) {
        const user = await this.userService.getByWallet(web3AuthDto.walletAddress);
        if (user.isDeleted) { throw new UnauthorizedException('Your account has been deleted. ...'); }
        if (user.isBlocked) { throw new UnauthorizedException('Your account has been blocked. ...'); }
        console.log(user, 'user')
        // Rotate the nonce now that it's been consumed - otherwise the
        // same signature stays valid forever and can be replayed to log
        // in again, which now also mints a fresh "wallet verified" claim
        // each time.
        await this.userService.updateNonce(web3AuthDto.walletAddress, generateNonce(32));
        await this.userService.updateTheUserSource(user.id, web3AuthDto.source);
        const response = await this.handleResponse(user.id, {
            walletAddress: web3AuthDto.walletAddress,
            walletVerifiedAt: Date.now()
        });
        response.user = user;
        this.eventEmitter.emit('reputation.login', { userId: user.id });
        return response;
    } else { ... }
}
```

There are five distinct changes in `web3-auth.service.ts` since `a131b429`.

The first three are the ethers v6 renames from `26b1f0e4` (14 September): `ethers.utils.getAddress` became `ethers.getAddress` in `login` (line 63), `authenticate` (line 82) and `verifyAndUpdateWalletAddress` (line 146), and `ethers.utils.verifyMessage` became `ethers.verifyMessage` in `decodeSignature` (line 124). Behaviour is identical, the functions just moved to the top level of the ethers namespace in v6.

The fourth is the nonce rotation at line 103, `await this.userService.updateNonce(web3AuthDto.walletAddress, generateNonce(32))`, right after the deleted and blocked checks pass. This closes the replay gap that [the web3 wallet auth note](../01-auth-and-identity/03-web3-wallet-auth.md) used to describe, where the same `(walletAddress, signature)` pair could be replayed to log in again because the stored nonce never changed. The comment explains why it became urgent: a replayed wallet login would now also mint a brand new "wallet verified" claim with a fresh timestamp each time, which would turn a replayed old signature into a way of refreshing wallet verification forever. Note the ordering: the rotation happens before `updateTheUserSource` and before tokens are issued, so if token issuing then fails, the nonce has still been consumed and the user simply has to request a new one, which is the safe failure direction.

The fifth is `handleResponse` (line 184), which gained an optional `verifiedWalletInfo?: VerifiedWalletInfo` parameter and passes it straight to `this.authService.getTokens(userId, verifiedWalletInfo)`. The only caller that passes it is `authenticate`. The Solana login path (`solanAuthenticate`) still calls `handleResponse(user.id)` with no proof, which is correct and important: as the web3 note explains, that path never verifies a signature at all, so it must never mint a wallet verified claim, and it does not.

What did not change: `verifyAndUpdateWalletAddress` (the `POST /web3-auth/verify` "link a wallet to my account" flow) still reads the nonce with `getNonceById` at line 147 and never rotates it afterwards, and it still does not mint any claim (it returns `void`, the user keeps their old tokens). The nonce replay gap therefore still exists there, although replaying it only re links a wallet the attacker could not control anyway, so the impact is low. And `getRefreshToken` at line 326 still calls `this.authService.refreshTokens(userId, refreshtokenDto.refreshToken)` with two arguments.

## The auth changes since baseline, line by line

```ts
// src/components/auth/auth.service.ts (line 188)
async refreshTokens(userId: string, refreshToken: string, verifiedWalletInfo?: VerifiedWalletInfo): Promise<TokenResponseDto> {
    const refreshTokenFromDB = await this.userService.getHashedRefreshTokenByUserId(userId);
    if (!refreshTokenFromDB) throw new UnauthorizedException('Access Denied');
    const refreshTokenMatches = await verifyHash(refreshTokenFromDB, refreshToken);
    if (!refreshTokenMatches) throw new UnauthorizedException('Access Denied');
    // Carry forward the wallet-verification claims from the token being rotated so a
    // refresh mid-session doesn't silently strip them and force RequireVerifiedWalletGuard
    // to reject the user until they re-sign with their wallet.
    const tokens = await this.getTokens(userId, verifiedWalletInfo);
    await this.userService.setCurrentRefreshToken(tokens.refreshToken, userId);
    return new TokenResponseDto(tokens.accessToken, tokens.refreshToken);
}
```

```ts
// src/components/auth/auth.controller.ts (lines 42 to 51)
@UseGuards(RefreshTokenGuard)
@Get('refresh-token')
async refreshTokens(@Req() req: Request): Promise<Response> {
    const userId = req.user['userId'];
    const refreshToken = req.user['refreshToken'];
    const walletAddress = req.user['walletAddress'];
    const walletVerifiedAt = req.user['walletVerifiedAt'];
    const verifiedWalletInfo = walletAddress && typeof walletVerifiedAt === 'number' ? { walletAddress, walletVerifiedAt } : undefined;
    return new Response(AuthSuccessMessage.TOKEN_REFRESHED, await this.authService.refreshTokens(userId, refreshToken, verifiedWalletInfo));
}
```

The full list of changes under `src/components/auth/` since baseline:

| File | Change |
|---|---|
| `interface/jwt-payload.interface.ts` | `JwtPayload` gains `walletAddress?` and `walletVerifiedAt?`; new `VerifiedWalletInfo` interface |
| `interface/auth.service.interface.ts` | imports `VerifiedWalletInfo`; `refreshTokens` and `getTokens` signatures gain the optional third and second parameter respectively |
| `auth.service.ts` | imports `VerifiedWalletInfo`; `refreshTokens` forwards the proof; `getTokens` conditionally spreads the claims into the payload |
| `auth.controller.ts` | `GET /auth/refresh-token` reads the two claims off the decoded refresh token and passes them on only when both are present and the timestamp is a number |
| `dto/refresh-token-validate-response.dto.ts` | two optional fields added |
| `auth.service.spec.ts` | new file, five tests (below) |

Two design points in the controller deserve a moment. First, the claims come from the refresh token itself, not the request body or the old access token, so they are as trustworthy as the refresh token's signature, a client cannot inject a wallet into a refresh. Second, the timestamp is copied, not reset. A refresh does not count as a new proof, otherwise any user could keep a wallet "verified" forever by refreshing every few minutes without ever touching the wallet again. Combined with the guard's freshness window, this means the window always counts from the last real signature.

## The prove wallet flow (B02b)

### The controller

```ts
// src/components/marketplacev2/wallet-verification/wallet-verification.controller.ts
/**
 * B-02b: prove-wallet is the only place in the product outside Seaport order
 * signing where a wallet-signature prompt appears. It covers two rows of the
 * login matrix: an email-login user with a wallet already linked but never
 * proved this session, and an email-login user with no wallet yet (connect
 * elsewhere, then prove here).
 */
@ApiTags('marketplacev2-wallet-verification')
@ApiBearerAuth('defaultBearerAuth')
@Controller('marketplacev2/orders/auth')
@UseGuards(AccessTokenGuard)
export class WalletVerificationController {
    constructor(
        @Inject('WalletVerificationServiceInterface')
        private readonly walletVerificationService: WalletVerificationServiceInterface
    ) {}

    @Get('wallet-nonce')
    public async getWalletNonce(@Req() req: Request): Promise<Response> {
        const userId = req.user['userId'];
        return new Response(WalletVerificationSuccessMessage.NONCE_GENERATED, await this.walletVerificationService.getWalletNonce(userId));
    }

    @Post('prove-wallet')
    public async proveWallet(@Body() proveWalletDto: ProveWalletDto, @Req() req: Request): Promise<Response> {
        const userId = req.user['userId'];
        const tokens = await this.walletVerificationService.proveWallet(userId, proveWalletDto);
        return new Response(WalletVerificationSuccessMessage.WALLET_PROVED, tokens);
    }
}
```

A textbook thin controller in the house style: `AccessTokenGuard` at class level (so both routes require a normal logged in session, no wallet claim needed yet, which is the whole point), the service injected by its string token, the `userId` taken from the decoded token and never from the body, and each method wrapping its result in `new Response(message, result)`. The two success messages come from a small enum:

```ts
// src/components/marketplacev2/wallet-verification/enum/wallet-verification-success-message.enum.ts
export enum WalletVerificationSuccessMessage {
    NONCE_GENERATED = 'Wallet Nonce Generated',
    WALLET_PROVED = 'Wallet Verified'
}
```

The module wiring is just as plain. `WalletVerificationModule` imports `AuthModule` (for `AuthServiceInterface.getTokens`), `UserModule` (for `setCurrentRefreshToken`), `WalletAddressModule` (for `WalletRepoInterface.findAllByUserId`) and `LoggerModule` (for `CustomLoggerService`), declares the controller, and binds `'WalletVerificationServiceInterface'` to `WalletVerificationService`. It exports nothing. The cache comes from the global `CacheModule.register({ isGlobal: true, ttl: 60000 })` in `app.module.ts:120`, which is why the module does not import anything for it.

### The DTOs

```ts
// src/components/marketplacev2/wallet-verification/dto/prove-wallet.dto.ts
export class ProveWalletDto {
    @ApiProperty({ description: 'The wallet address the caller claims to control. Eg: MetaMask Wallet' })
    @IsNotEmpty()
    @IsString()
    walletAddress: string;

    @ApiProperty({ description: 'Signature over buildWalletSignMessage(nonce), produced by the wallet named above' })
    @IsNotEmpty()
    @IsString()
    signature: string;

    @ApiProperty({ description: 'The single-use nonce returned by GET /marketplacev2/orders/auth/wallet-nonce' })
    @IsNotEmpty()
    @IsString()
    nonce: string;
}
```

| DTO | Field | Validators | Meaning |
|---|---|---|---|
| `ProveWalletDto` (request body) | `walletAddress` | `@IsNotEmpty()`, `@IsString()` | the address the caller says signed, any casing |
| | `signature` | `@IsNotEmpty()`, `@IsString()` | the `personal_sign` signature over `buildWalletSignMessage(nonce)` |
| | `nonce` | `@IsNotEmpty()`, `@IsString()` | the nonce returned by `wallet-nonce` |
| `WalletNonceResponseDto` (response `result`) | `nonce` | none, built by the constructor | 64 hex characters from `generateNonce(32)` |
| | `expiresInMs` | none | always `300000`, five minutes |

Because the global `ValidationPipe` runs with `whitelist: true`, any extra property in the prove body is stripped. Notice there is no `@IsEthereumAddress()` on `walletAddress` and no hex format check on `signature`, malformed values are caught later by the service (as a 401), not by validation (as a 400).

### The service, step by step

```ts
// src/components/marketplacev2/wallet-verification/wallet-verification.service.ts
/**
 * B-02b prove-wallet flow. Deliberately does NOT call web3-auth's
 * verifyAndUpdateWalletAddress: that method reads/writes tbl_wallet_address.nonce
 * (via UserService.getNonceById/updateUserWallet), which is the exact login-nonce
 * column the brief says this flow must never touch. What IS reused verbatim is
 * buildWalletSignMessage (the signed message format) and WalletRepoInterface.findAllByUserId
 * (confirming wallet ownership) - the nonce storage and consumption below is the
 * genuinely new piece, on top of the cache manager already wired globally in app.module.ts.
 */
@Injectable()
export class WalletVerificationService implements WalletVerificationServiceInterface {
    private static readonly NONCE_TTL_MS = 5 * 60 * 1000;
    private static readonly CACHE_KEY_PREFIX = 'marketplacev2:prove-wallet-nonce:';

    async getWalletNonce(userId: string): Promise<WalletNonceResponseDto> {
        const nonce = generateNonce(32);
        await this.cache.set(this.cacheKey(userId), nonce, WalletVerificationService.NONCE_TTL_MS);
        return new WalletNonceResponseDto(nonce, WalletVerificationService.NONCE_TTL_MS);
    }

    async proveWallet(userId: string, proveWalletDto: ProveWalletDto): Promise<TokenResponseDto> {
        const logger = new Logger(WalletVerificationService.name + '-proveWallet');
        const key = this.cacheKey(userId);
        const storedNonce = await this.cache.get<string>(key);

        if (!storedNonce || storedNonce !== proveWalletDto.nonce) {
            logger.error(WalletVerificationErrorMessage.NONCE_EXPIRED_OR_INVALID);
            throw new BadRequestException(WalletVerificationErrorMessage.NONCE_EXPIRED_OR_INVALID);
        }
        // Consume immediately so a replayed request (even in-flight/concurrent) always misses.
        await this.cache.del(key);

        let recoveredAddress: string;
        try {
            recoveredAddress = ethers.verifyMessage(buildWalletSignMessage(proveWalletDto.nonce), proveWalletDto.signature);
        } catch (err) {
            logger.error(err);
            this.customLoggerService.error(`${err}`);
            throw new UnauthorizedException(WalletVerificationErrorMessage.SIGNATURE_INVALID);
        }

        if (recoveredAddress.toLowerCase() !== proveWalletDto.walletAddress.toLowerCase()) {
            logger.error(WalletVerificationErrorMessage.SIGNATURE_INVALID);
            throw new UnauthorizedException(WalletVerificationErrorMessage.SIGNATURE_INVALID);
        }

        const userWallets = await this.walletRepo.findAllByUserId(userId);
        const belongsToUser = userWallets.some((wallet) => wallet.walletAddress?.toLowerCase() === recoveredAddress.toLowerCase());
        if (!belongsToUser) {
            logger.error(WalletVerificationErrorMessage.WALLET_NOT_LINKED_TO_USER);
            throw new ForbiddenException(WalletVerificationErrorMessage.WALLET_NOT_LINKED_TO_USER);
        }

        const checksummedAddress = ethers.getAddress(recoveredAddress);
        const tokens = await this.authService.getTokens(userId, {
            walletAddress: checksummedAddress,
            walletVerifiedAt: Date.now()
        });
        await this.userService.setCurrentRefreshToken(tokens.refreshToken, userId);
        return tokens;
    }
}
```

Read the long doc comment first, because it is the design decision. The existing "link a wallet" flow in `web3-auth` stores its nonce in the `nonce` column of `tbl_wallet_address`, the very same column the wallet login flow reads. If prove wallet reused that column, then asking for a prove nonce would overwrite the login nonce, and a user who had a wallet login half finished in another tab would see it fail. So this flow stores its nonce somewhere else entirely, in the Nest cache, under the key `marketplacev2:prove-wallet-nonce:<userId>`, with a five minute TTL. It reuses two existing pieces on purpose: `buildWalletSignMessage` from `src/@core/utils/helper.ts` (so the user sees the same "Sign this message to verify wallet ownership on Endless Domains." text, and the frontend's matching function in `webui/src/core/services/wallet.service.ts` works unchanged), and `findAllByUserId` from `wallet.repo.ts` (so "which wallets belong to this user" has one source of truth).

`getWalletNonce` generates 32 random bytes as 64 hex characters, stores it with `cache.set(key, value, ttl)` (with `cache-manager` version 7, the third argument is milliseconds, so 300,000 really is five minutes, in older versions it was seconds and this would have been 83 hours), and returns both the nonce and its lifetime. Because the key is per user, calling it again simply replaces the previous nonce.

`proveWallet` then runs five gates in order:

1. Nonce lookup. If there is no nonce for this user, or it differs from the one in the body, throw `BadRequestException` (400) with `NONCE_EXPIRED_OR_INVALID`. This covers expired, never issued, already used, and "you asked for a newer nonce since."
2. Consume. `cache.del(key)` immediately, before checking the signature at all. The intent is that a nonce can be tried once, success or failure.
3. Signature recovery. `ethers.verifyMessage(message, signature)` recovers the address that signed the exact message text. A structurally malformed signature makes ethers throw, which becomes `UnauthorizedException` (401) with `SIGNATURE_INVALID`, and is logged to both the Nest logger and CloudWatch through `CustomLoggerService`.
4. Address match. The recovered address must equal the claimed `walletAddress`, compared lowercase so casing does not matter. A valid signature over a different message, or by a different key, recovers some other address and fails here with the same 401.
5. Ownership. The recovered address must be one of the user's rows in `tbl_wallet_address`. If not, `ForbiddenException` (403) with `WALLET_NOT_LINKED_TO_USER`. This is what makes the proof meaningful: proving you control a random wallet is not enough, it has to be a wallet already attached to this account (through `POST /web3-auth/verify` or a wallet registration).

Only then does it mint a fresh token pair with `{ walletAddress: ethers.getAddress(recoveredAddress), walletVerifiedAt: Date.now() }`, store the new refresh token hash with `setCurrentRefreshToken` (which also means the previous refresh token stops working, the usual rotation), and return `{ accessToken, refreshToken }` as the `result`. The address in the claim is always the `EIP-55` checksummed form, which matters because the order service compares offerers with strict equality against `getAddress(walletAddress)`.

The three error messages:

```ts
// src/components/marketplacev2/wallet-verification/enum/wallet-verification-error-message.enum.ts
export enum WalletVerificationErrorMessage {
    NONCE_EXPIRED_OR_INVALID = 'Nonce is invalid, expired, or already used. Request a new one.',
    SIGNATURE_INVALID = 'Signature does not match the claimed wallet address.',
    WALLET_NOT_LINKED_TO_USER = 'This wallet is not linked to your account.'
}
```

| Situation | Status | Message |
|---|---|---|
| No access token, or expired | 401 | Passport's default `Unauthorized` |
| Missing or empty body field | 400 | class validator message array |
| Nonce missing, expired, mismatched or reused | 400 | `NONCE_EXPIRED_OR_INVALID` |
| Signature unparseable, or recovers to a different address | 401 | `SIGNATURE_INVALID` |
| Wallet not linked to this account | 403 | `WALLET_NOT_LINKED_TO_USER` |
| Success | 200/201 | `Wallet Verified`, `result: { accessToken, refreshToken }` |

`POST` routes in Nest default to status 201, so a successful `prove-wallet` returns 201, not 200, worth knowing if your fetch wrapper checks for exactly 200.

The interface file documents both methods in one line each, and is worth quoting because it states the contract clearly:

```ts
// src/components/marketplacev2/wallet-verification/interface/wallet-verification.service.interface.ts
export interface WalletVerificationServiceInterface {
    /** Issues a short-lived, single-use nonce bound to this session's userId. Stored separately from tbl_wallet_address.nonce. */
    getWalletNonce(userId: string): Promise<WalletNonceResponseDto>;

    /** Verifies the signature over the nonce, confirms the wallet belongs to this user, consumes the nonce, and re-issues the access/refresh tokens with the wallet attached. */
    proveWallet(userId: string, proveWalletDto: ProveWalletDto): Promise<TokenResponseDto>;
}
```

## `RequireVerifiedWalletGuard`

```ts
// src/@core/common/guards/require-verified-wallet.guard.ts
import { CanActivate, ExecutionContext, ForbiddenException, Injectable } from '@nestjs/common';
import { WalletVerificationErrorCode } from '../enum/wallet-verification-error-code.enum';

// JWT_ACCESS_TOKEN_EXPIRATION is 1h (confirmed via the AWS Secrets Manager
// secret this env's ConfigService actually resolves it from - .env.sample's
// "3d" is stale and not what's served at runtime).
//
// B02_SPRINT_PLAN.md's brief called for this window to be kept meaningfully
// shorter than the token's own lifetime, specifically so a stale wallet
// verification can't ride along and authorize actions for the token's full
// life - Sprint 2 originally set this to 15 minutes for that reason. That
// guidance was deliberately overridden on 2026-09-15 at product's request:
// this now matches JWT_ACCESS_TOKEN_EXPIRATION exactly, so a wallet
// verification is considered fresh for as long as the token itself is
// valid. If that decision changes, restore a window meaningfully shorter
// than the token's lifetime rather than picking an arbitrary value.
export const WALLET_VERIFICATION_WINDOW_MS = 60 * 60 * 1000;

@Injectable()
export class RequireVerifiedWalletGuard implements CanActivate {
    canActivate(context: ExecutionContext): boolean {
        const request = context.switchToHttp().getRequest();
        const user = request.user as Express.User | undefined;
        const walletAddress = user?.walletAddress;
        const walletVerifiedAt = user?.walletVerifiedAt;

        const isVerified =
            !!walletAddress &&
            typeof walletVerifiedAt === 'number' &&
            Date.now() - walletVerifiedAt <= WALLET_VERIFICATION_WINDOW_MS;

        if (!isVerified) {
            throw new ForbiddenException({
                code: WalletVerificationErrorCode.WALLET_NOT_VERIFIED,
                message: 'Wallet verification is missing or has expired for this session.'
            });
        }

        request.walletAddress = walletAddress;
        return true;
    }
}
```

```ts
// src/@core/common/enum/wallet-verification-error-code.enum.ts
export enum WalletVerificationErrorCode {
    WALLET_NOT_VERIFIED = 'WALLET_NOT_VERIFIED'
}
```

The guard is synchronous and pure, it never touches the database, the cache or the network. It reads `request.user`, which only exists because `AccessTokenGuard` already ran and decoded the token, so it must always be listed after `AccessTokenGuard` in `@UseGuards(...)`. If someone ever applied it on its own, `request.user` would be `undefined` and the caller would get a 403 `WALLET_NOT_VERIFIED` instead of the more accurate 401, misleading but not unsafe. It passes only when three things are true: there is a non empty `walletAddress` claim, `walletVerifiedAt` is a number, and that timestamp is no more than one hour old (the comparison is `<=`, so exactly one hour still passes). On success it copies the address onto `request.walletAddress` and returns `true`.

### The long comment and the 15 September decision

The comment above `WALLET_VERIFICATION_WINDOW_MS` is a small piece of engineering history, and worth understanding because it is the most important security parameter in this whole flow. The original B02 sprint plan said the freshness window should be meaningfully shorter than the access token's lifetime. The reasoning: a JWT cannot be revoked, so if wallet verification were valid for the token's whole life, a stolen or forgotten token would carry wallet authority for that entire time, and a user who unlinked a wallet or handed their laptop to someone would still have a "verified" session. Sprint 2 therefore shipped a fifteen minute window, meaning a user would be asked to sign again every fifteen minutes before marketplace actions.

On 15 September 2026, product overrode that (presumably because a signature prompt every fifteen minutes is a poor experience), and the window became one hour, which the comment says matches `JWT_ACCESS_TOKEN_EXPIRATION` in the real AWS secret. The comment also records that `.env.sample`'s `JWT_ACCESS_TOKEN_EXPIRATION=3d` is stale. In practice, today, with a one hour access token, the window is close to meaningless as a separate control: a token that has the claim and is still valid is almost always within the window. Where it still bites is after a refresh, because the refresh carries the original timestamp forward. If you prove at 10:00 and refresh at 10:55, the new access token is valid until 11:55, but the guard will start rejecting it at 11:00, and you will need to prove again. And if anyone ever lengthens the access token lifetime in the secret without touching this constant, the window quietly becomes the tighter control again, which is the behaviour the comment asks future maintainers to preserve.

### The error shape the frontend sees

Because the guard passes an object to `ForbiddenException`, Nest's default exception handler uses that object as the response body verbatim. There is no global exception filter in this app, so the 403 body is exactly:

```ts
// HTTP 403 body from RequireVerifiedWalletGuard
{ "code": "WALLET_NOT_VERIFIED", "message": "Wallet verification is missing or has expired for this session." }
```

Note that it has no `statusCode` or `error` field, unlike Nest's default string exception bodies (`{ statusCode: 403, message: '...', error: 'Forbidden' }`). That is actually the useful part: `code` is a stable machine readable value the frontend can branch on, while the `WALLET_NOT_LINKED_TO_USER` 403 from `prove-wallet` uses the default shape with no `code`. This is the first place in the codebase with an explicit error code enum, worth copying if more stable codes are needed later.

### Where the guard is applied

Every route that acts on a wallet uses `@UseGuards(AccessTokenGuard, RequireVerifiedWalletGuard)`: `POST /marketplacev2/orders` (create), `POST /marketplacev2/orders/:orderHash/cancel`, `GET /marketplacev2/orders/my-listings`, `GET /marketplacev2/orders/my-purchases`, `GET /marketplacev2/orders/listing-history`, `GET /marketplacev2/orders/transactions/mine`, plus the three admin only `_health/wallet-verified`, `_health/validate-local` and `_health/validate-chain` routes (with `AdminTokenGuard` in between). Routes that only need the user, not the wallet, deliberately skip it: `my-domains`, the watchlist routes and `listing-status/sync` each carry a comment explaining they act on `userId`, not on a chain action.

## The Express type augmentation, and the one rule

```ts
// src/@types/express/index.d.ts
declare namespace Express {
    interface User {
        userId: string;
        refreshToken?: string;
        walletAddress?: string;
        walletVerifiedAt?: number;
    }

    interface Request {
        // Set by RequireVerifiedWalletGuard. The only source handlers should
        // read a verified wallet address from - never re-looked-up from
        // userId, never trusted from the request body.
        walletAddress?: string;
    }
}
```

This file existed at baseline with just `userId` and `refreshToken?`. It uses TypeScript's declaration merging: Passport's types declare an empty global `Express.User` interface and Express declares `Express.Request`, and any later `declare namespace Express { interface User { ... } }` merges extra fields into them. That is how `req.user.walletAddress` and `req.walletAddress` type check everywhere without casts. The two additions to `User` describe what the strategies put on `req.user`. The one addition to `Request` describes what the guard puts on the request.

The comment on `Request.walletAddress` is the rule this whole design depends on, and it is followed everywhere in the current code: a handler that needs the verified wallet reads `req.walletAddress`, and nothing else. Not `req.user.walletAddress` (which has not been freshness checked), not a lookup by `userId` (a user can have several wallets, and only one was proved), and never `dto.walletAddress` or `dto.parameters.offerer` from the body (which the client controls). For example:

```ts
// src/components/marketplacev2/order/order.controller.ts
@Post()
@UseGuards(AccessTokenGuard, RequireVerifiedWalletGuard)
async createOrder(@Body() dto: CreateOrderDto, @Req() req: Request): Promise<Response> {
    return new Response('Order created', await this.orderService.create(dto, req.walletAddress, req.user['userId']));
}
```

and inside `validateLocal`, check 1 (`OFFERER_NOT_CALLER`) is exactly `components.offerer !== getAddress(walletAddress)`, the comparison that ties the signed order to the proved session. A search of the non spec v2 code finds no handler reading `req.user['walletAddress']` directly.

## Every spec test, by ID

### `wallet-verification.service.spec.ts`

The helper `buildService` builds the service with a hand rolled cache backed by a real `Map` (so `set`, `get` and `del` really interact), a wallet repo mock returning `{ walletAddress }` rows for the addresses you pass, an auth service mock whose `getTokens` resolves `{ accessToken: 'access-token', refreshToken: 'refresh-token' }`, and a user service mock. The signing tests use real ethers wallets (`ethers.Wallet.createRandom()`) and real signatures over `buildWalletSignMessage(nonce)`, so the cryptography is not mocked at all, which is a good choice.

| ID | Test | What it proves |
|---|---|---|
| TC3.1 | `getWalletNonce stores a nonce bound to the session user with a short expiry` | the nonce is a string, `expiresInMs > 0`, and `cache.set` was called with `marketplacev2:prove-wallet-nonce:user-1`, the nonce, and the TTL |
| TC3.2 | `valid signature over the correct message + fresh nonce re-issues tokens with the wallet attached` | tokens are returned, `getTokens` got `walletAddress` and a numeric `walletVerifiedAt`, the refresh token was stored, and the nonce was deleted |
| TC3.3 | `replaying the same nonce a second time is rejected` | the same valid request sent twice in sequence fails the second time with `BadRequestException` |
| TC3.4 | `a nonce that was never issued (expired/unknown) is rejected` | with nothing in the cache, a request fails with `BadRequestException` |
| TC3.5 | `valid signature but the recovered address is not linked to this user is rejected` | a correct signature from an unlinked wallet fails with `ForbiddenException` |
| TC3.6 | `signed message not matching buildWalletSignMessage(nonce) fails recovery` | a signature over `'some other message entirely'` fails with `UnauthorizedException` |

TC3.3 only tests sequential replay. There is no test for two concurrent requests, which is exactly where the implementation has a gap (see risks).

### `require-verified-wallet.guard.spec.ts`

| ID | Test | What it proves |
|---|---|---|
| TC2.1 | `throws WALLET_NOT_VERIFIED when walletAddress is absent` | a user with only `userId` gets `ForbiddenException` whose response body matches `{ code: 'WALLET_NOT_VERIFIED' }` |
| TC2.2 | `throws WALLET_NOT_VERIFIED when walletVerifiedAt is older than the window` | one hour plus one second old is rejected |
| TC2.3 | `passes through and attaches req.walletAddress for a fresh, valid claim` | returns `true` and sets `request.walletAddress` to `'0xabc'` |
| TC2.5 | `window edge (exact boundary) passes` | a timestamp exactly `WALLET_VERIFICATION_WINDOW_MS` old passes |
| TC2.5 | `window edge + 1 second fails` | one second past the window fails |

There is no TC2.4 in this file. The `OrderController` doc comment mentions "TC2.1/TC2.4" being exercised over HTTP via the `_health` routes, so TC2.4 appears to be the manual QA test through `_health/wallet-verified` rather than a unit test. More importantly, the "exact boundary passes" test is timing sensitive. It computes `walletVerifiedAt = Date.now() - WINDOW` when building the context, and the guard calls `Date.now()` again a moment later. If even one millisecond ticks over between those two calls (plausible on a loaded CI runner), the difference becomes `WINDOW + 1`, which is greater than the window, and the test fails. Using `jest.useFakeTimers()` with `jest.setSystemTime(...)`, or injecting a clock, would make it deterministic.

### `auth.service.spec.ts` (new file)

It mocks `verifyHash` from `src/@core/utils/helper` (keeping the real implementations of everything else via `jest.requireActual`) and builds `AuthService` by passing thirteen positional constructor arguments, most as `{}`. The `jwtService.signAsync` mock returns `signed:` plus the JSON payload, so tests can inspect exactly what was signed.

| Test | What it proves |
|---|---|
| `throws when there is no stored refresh token hash for the user` | `getHashedRefreshTokenByUserId` returning `null` gives `UnauthorizedException` |
| `throws when the supplied refresh token does not match the stored hash` | `verifyHash` false gives `UnauthorizedException` |
| `issues fresh tokens with no wallet claims when the caller has none to carry forward (unchanged behaviour)` | the signed payload is exactly `{ userId: 'user-1' }` |
| `carries the wallet verification claims forward into the newly issued tokens instead of dropping them` | both the access and refresh payloads equal `{ userId, walletAddress: '0xabc123', walletVerifiedAt: 1_700_000_000_000 }`, and the new refresh token is stored |
| `does not extend the wallet verification window on refresh` | the new access payload's `walletVerifiedAt` equals the original timestamp, not a new one |

These tests cover the service, not the controller, so the controller's own condition (`walletAddress && typeof walletVerifiedAt === 'number'`) is untested. The thirteen positional arguments are brittle: adding or reordering a constructor dependency in `AuthService` will break this spec in a confusing way. A `Test.createTestingModule` with named providers would be sturdier.

### `web3-auth.service.spec.ts` (new file)

Titled "Web3AuthService.decodeSignature (ethers.verifyMessage rename regression)," added in `26b1f0e4` with the ethers migration. It calls the private `decodeSignature` through `(service as any)`.

| Test | What it proves |
|---|---|
| `recovers the exact wallet address that signed the nonce message` | a real signature over `buildWalletSignMessage(nonce)` recovers `wallet.address` |
| `throws UnauthorizedException for a malformed/tampered signature` | a signature truncated by one hex character makes ethers throw, mapped to `UnauthorizedException`; the comment explains why truncation (wrong length) rather than flipping a byte (which would still recover some address) |
| `throws UnauthorizedException when the signature was produced over a different nonce` | despite the title, it asserts `not.toThrow()` and that the recovered address differs from the signer, with a comment explaining that the mismatch is caught by the callers, not `decodeSignature` |

That third title contradicts its own assertions, which will confuse whoever reads a failing test report. Renaming it to something like "recovers a different address when signed over a different nonce" would match what it checks. None of the three tests covers the new nonce rotation in `authenticate` or the new claims passed to `handleResponse`.

## The frontend sequence, end to end

Here is what the React app has to do, in order. Every path is under `/api/v1`, and every response body is the house envelope `{ message, result }` (the `success` and `statusCode` fields on `Response` are declared but never populated by these controllers, so do not rely on them).

### A user who logs in with their wallet

1. `GET /web3-auth/login/:walletAddress`, no auth. `result.nonce` is the login nonce.
2. Ask the wallet to `personal_sign` the exact text from `buildWalletSignMessage(nonce)`: `Sign this message to verify wallet ownership on Endless Domains.` followed by a blank line and `Nonce: <nonce>`. Use the shared frontend helper so the text is identical.
3. `POST /web3-auth/authenticate` with body `{ walletAddress, signature, network, source }`, where `source` must be `main_site` or `marketplace` (`@IsIn` on `Web3AuthDto`) and `network` is the EVM network string (`evm_network` is what `login` stores for new users). The returned `accessToken` already carries `walletAddress` and `walletVerifiedAt`. No prove step is needed for the next hour.

### A user who logs in with email (or Google)

1. `POST /auth/login` as before. The tokens have no wallet claims.
2. Connect the wallet in the browser and check whether its address is already linked to the account. If it is not, link it first through the existing flow: `GET /web3-auth/generate-nonce` with `Authorization: Bearer <accessToken>`, sign `buildWalletSignMessage(nonce)`, then `POST /web3-auth/verify` with `{ walletAddress, signature, network, source }` and the same header. That links the wallet but does not change the tokens.
3. `GET /marketplacev2/orders/auth/wallet-nonce` with `Authorization: Bearer <accessToken>`. You get `result: { nonce, expiresInMs: 300000 }`. Start a five minute timer if you like.
4. Ask the wallet to sign `buildWalletSignMessage(nonce)`, the same text as login.
5. `POST /marketplacev2/orders/auth/prove-wallet` with the same header and body `{ walletAddress, signature, nonce }`. Expect status 201 and `result: { accessToken, refreshToken }`.
6. Replace both stored tokens immediately. The old refresh token no longer works, because the new one's hash has replaced it on the user row.

### Using the marketplace

Send `Authorization: Bearer <accessToken>` on every v2 call that needs it. Before showing "List for sale," "Cancel listing," "My listings," "My purchases," "Listing history" or "My transactions," you can decode the access token on the client (it is a plain JWT, base64url decode the middle part) and check that `walletAddress` exists and `Date.now() - walletVerifiedAt` is under one hour (it is in milliseconds), and that `walletAddress` matches the account currently selected in the wallet extension. If not, run steps 3 to 6 before the action rather than after a failure. When building a Seaport order, the `offerer` must be that same address, or check 1 rejects it with `OFFERER_NOT_CALLER`.

### Refreshing

Use `GET /auth/refresh-token` with `Authorization: Bearer <refreshToken>`. It keeps the wallet claims and their original timestamp. Avoid `POST /web3-auth/refresh-token`, which currently drops the claims and also returns an empty body instead of an error when something goes wrong.

### Error handling cheat sheet

| What you get | What it means | What to do |
|---|---|---|
| 403 with `code: 'WALLET_NOT_VERIFIED'` | no wallet claim, or older than one hour | run the prove flow (steps 3 to 6), then retry the original request once with the new token |
| 400 `Nonce is invalid, expired, or already used. Request a new one.` | five minutes passed, a second tab requested a newer nonce, or this nonce was already tried | fetch a fresh nonce and sign again |
| 401 `Signature does not match the claimed wallet address.` | the user signed with a different account than the one you sent, or the message text differs | prompt the user to switch accounts in the wallet, check the message helper |
| 403 `This wallet is not linked to your account.` (no `code` field) | the signing wallet is not attached to this user | run the link flow (`generate-nonce` then `verify`) first |
| 401 `Unauthorized` | the access token itself is missing or expired | refresh, then retry |

Tell the two 403s apart by the presence of `code`. And because each failed prove attempt consumes the nonce (step 2 of the service deletes it before checking the signature), always fetch a new nonce before retrying a failed prove.

## Bugs, gaps and risks

1. `src/components/web3-auth/web3-auth.service.ts:329` calls `refreshTokens(userId, refreshtokenDto.refreshToken)` without the wallet info, so a frontend using `POST /web3-auth/refresh-token` silently loses the wallet claims and the next marketplace call fails with `WALLET_NOT_VERIFIED`. The surrounding `try/catch` also swallows errors and returns `undefined`, so a bad refresh token produces a 201 with an empty body instead of a 401.
2. `src/components/marketplacev2/wallet-verification/wallet-verification.service.ts:51` to `:58` reads then deletes the nonce in two separate awaits, so two concurrent prove requests with the same nonce can both pass the check before either delete lands, contrary to the comment claiming "even in flight/concurrent" replays miss. Impact is low (both requests are the same user with a valid signature), but the comment overstates the guarantee, an atomic "get and delete" or a Redis `GETDEL` would make it true.
3. `src/app.module.ts:120` registers the cache as the default in memory store, so prove nonces live in one Node process. Any deployment with more than one instance behind a load balancer (the infrastructure notes describe several ECS tasks) will intermittently fail `prove-wallet` with `NONCE_EXPIRED_OR_INVALID` when the nonce and prove requests land on different instances, and every deploy or PM2 restart wipes outstanding nonces.
4. `src/@core/common/guards/require-verified-wallet.guard.ts:27` to `:30` checks only the claim's age, not whether the wallet is still linked, so a wallet unlinked (or an account whose wallet changed hands) still authorizes create and cancel for up to an hour. On chain ownership checks in order validation limit the damage for create, but `my-listings`, `my-purchases` and `transactions/mine` would keep serving that wallet's data.
5. `src/@core/common/guards/require-verified-wallet.guard.ts:17` makes the window equal to the access token lifetime after the 15 September product override, so the window adds almost no protection beyond the token's own expiry, and the comment's claim that the token lives one hour contradicts `.env.sample:29` (`JWT_ACCESS_TOKEN_EXPIRATION=3d`), which is stale.
6. `src/@core/common/guards/require-verified-wallet.guard.spec.ts:47` to `:54`, the exact boundary test, is flaky because `Date.now()` is read twice and one elapsed millisecond makes it fail.
7. `src/components/web3-auth/web3-auth.service.spec.ts:40` has a title saying it "throws UnauthorizedException" while the test asserts it does not throw.
8. `src/components/web3-auth/web3-auth.service.ts:147`, `verifyAndUpdateWalletAddress` still never rotates the nonce after use, unlike `authenticate` now does, so the same link signature can be replayed (low impact).
9. `src/components/marketplacev2/wallet-verification/wallet-verification.service.ts:44`, one nonce key per user, means a second tab or a double click requesting a nonce invalidates the first, producing a confusing 400 for a signature the user just made.
10. `src/components/marketplacev2/wallet-verification/wallet-verification.service.ts:58` consumes the nonce before verifying the signature, so a malformed request (for example a wallet extension returning a bad signature) burns the nonce and forces a new prompt.
11. `src/components/wallet-address/wallet.repo.ts:66`, `findAllByUserId`, does not filter on `isDeleted` even though `WalletAddressEntity` extends `BaseEntity`, so if wallets are ever soft deleted, a removed wallet could still be proved.
12. `src/components/marketplacev2/wallet-verification/dto/prove-wallet.dto.ts:7` validates `walletAddress` only as a non empty string, not `@IsEthereumAddress()`, so obviously malformed input becomes a 401 from deep in the service instead of a clear 400.
13. `src/components/auth/auth.service.spec.ts:26` constructs `AuthService` with thirteen positional arguments, which will break confusingly on any constructor change.

## Why this is a good pattern to learn

Strip away the details and the idea is simple and reusable: when an action needs a stronger proof than "logged in," do not bolt a database flag onto the user, put a signed, timestamped claim into the session token, check its freshness in a tiny guard, and give handlers exactly one trusted place to read the proved value from. You will see the same shape in "step up authentication" for banking apps (re enter your password before a transfer) and in "sudo mode" on GitHub. The specific implementation here is careful in the right places (separate nonce storage, checksum normalisation, timestamp not bumped on refresh, nonce rotation on wallet login) and has a few rough edges (in memory nonce storage, one legacy refresh route, non atomic consume) that are easy to fix once you know where to look.
