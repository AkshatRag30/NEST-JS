# 03. Web3 Wallet Authentication

## The general idea: prove ownership by signing a nonce

Wallet based login cannot use a password, so it uses a signature challenge instead. The backend hands the frontend a random, single use looking value (a nonce), the frontend asks the user's wallet (MetaMask, Phantom, whatever) to sign a message containing that nonce, and the backend checks that the signature it gets back was produced by the private key belonging to the wallet address the user claims to own. Only the true holder of that wallet's private key can produce a valid signature, so a verified signature is treated as proof of ownership. That is the theory this module is built on, and for the EVM (Ethereum style) path, it is implemented correctly. For the Solana path, as this note explains in detail, it is not actually implemented at all.

## The nonce and the message it gets wrapped in

```ts
// src/@core/utils/helper.ts
export function generateNonce(size: number): string {
    return crypto.randomBytes(size).toString('hex');
}

// Wraps the raw nonce in non-hex text before it gets signed. A bare hex string
// is ambiguous to some wallet signing implementations (Trust Wallet in particular),
// which can decode it as raw bytes instead of UTF-8 text and sign a different
// message than ethers.utils.verifyMessage expects. This text must match the
// frontend's buildWalletSignMessage in webui/src/core/services/wallet.service.ts exactly.
export function buildWalletSignMessage(nonce: string): string {
    return `Sign this message to verify wallet ownership on Endless Domains.\n\nNonce: ${nonce}`;
}
```

`generateNonce(32)` produces 32 random bytes rendered as a 64 character hex string, using Node's cryptographically secure `crypto.randomBytes`, not `Math.random()`, which matters since a guessable nonce would undermine the whole scheme. The comment on `buildWalletSignMessage` is worth reading closely because it names a real, specific interoperability problem this team already ran into, some wallets sign a bare hex string as raw bytes rather than as UTF-8 text, which would silently produce a signature that `ethers.utils.verifyMessage` could never match, so the nonce gets wrapped in a fixed sentence before it is ever signed. The comment also flags that this exact string has to be kept in sync with a matching function in the separate frontend codebase, `webui/src/core/services/wallet.service.ts`, a real cross repository coupling that is easy to lose track of if the two projects are ever touched by people who do not know about each other's code.

## The EVM login flow, correctly verified

```ts
// src/components/web3-auth/web3-auth.service.ts
async login(walletAddress: string): Promise<NonceResponseDto> {
    walletAddress = ethers.utils.getAddress(walletAddress);
    const nonce = generateNonce(32);
    try {
        await this.userService.updateNonce(walletAddress, nonce);
        return new NonceResponseDto(nonce);
    } catch (err: any) {
        if (err instanceof HttpException && err.getStatus() === HttpStatus.NOT_FOUND) {
            await this.registerUser(walletAddress, nonce, 'evm_network');
            return new NonceResponseDto(nonce);
        }
        throw new Error(err);
    }
}

async authenticate(web3AuthDto: Web3AuthDto): Promise<ReturnLoginDto> {
    web3AuthDto.walletAddress = ethers.utils.getAddress(web3AuthDto.walletAddress);
    const nonce = await this.userService.getNonceByWalletAddress(web3AuthDto.walletAddress, web3AuthDto.network);
    const decodedAddress = this.decodeSignature(web3AuthDto.signature, nonce);
    if (web3AuthDto.walletAddress.toLowerCase() === decodedAddress.toLowerCase()) {
        const user = await this.userService.getByWallet(web3AuthDto.walletAddress);
        if (user.isDeleted) { throw new UnauthorizedException('...deleted...'); }
        if (user.isBlocked) { throw new UnauthorizedException('...blocked...'); }
        const response = await this.handleResponse(user.id);
        response.user = user;
        return response;
    } else {
        throw new UnauthorizedException(Web3AuthErrorMessage.INVALID_TOKEN);
    }
}

private decodeSignature(signature: string, nonce: string): string {
    try {
        return ethers.utils.verifyMessage(buildWalletSignMessage(nonce), signature);
    } catch (err) {
        throw new UnauthorizedException(Web3AuthErrorMessage.INVALID_TOKEN);
    }
}
```

`GET /web3-auth/login/:walletAddress` is the first step, it normalizes the address to its canonical checksummed form with `ethers.utils.getAddress` (so the same wallet cannot accidentally be treated as two different accounts due to letter casing), generates a nonce, and either stores it against an existing wallet record or, if the address has never been seen before, registers a brand new user around it via `registerUser`, still before any signature has been checked, an account row can exist purely from a wallet address showing up here. `POST /web3-auth/authenticate` is the second step, it looks the stored nonce back up by wallet address, calls `decodeSignature`, and `ethers.utils.verifyMessage(message, signature)` is the actual cryptographic check, it recovers the Ethereum address that must have signed the given message to produce that exact signature, and the whole check succeeds only if that recovered address matches the address the caller claims. This is real, correct EVM signature verification.

What this flow does not do is invalidate or rotate the nonce after a successful login. Nothing in `authenticate` calls `updateNonce` or any equivalent, the exact same nonce that was just used to log in successfully remains stored against that wallet's row until the next time someone calls `login` (or `generateAndSaveNonce`) to request a fresh one. A signed message and its signature, once produced, are not secret by nature, they exist as plain values on the wire between the frontend and this backend, and if a captured `(walletAddress, signature)` pair from that traffic were ever replayed against `POST /web3-auth/authenticate` a second time, the exact same nonce would still be sitting in the database and the exact same signature would still verify against it, since `ethers.utils.verifyMessage` is a pure, stateless check with no concept of "already used." The standard fix for this class of problem, generating a new nonce (or otherwise marking the old one consumed) the moment a nonce is successfully used, is present in this same file for the RBAC 2FA flow's TOTP handling in spirit but is not applied here. This is worth being precise about: it is a replay window, not an open door, an attacker still needs to have captured a real signed message from a real login attempt in the first place, over HTTPS that is a meaningfully high bar, but it is a real gap between this implementation and how nonce based signature auth is normally expected to behave.

## Adding or changing a wallet on an existing, already logged in account

```ts
@Post('verify')
@UseGuards(AccessTokenGuard)
public async verify(@Body() web3AuthDto: Web3AuthDto, @Req() req: Request): Promise<Response> {
    const userId = req.user['userId'];
    return new Response(Web3AuthSuccessMessage.WALLET_ADDRESS_VERIFIED, await this.web3AuthService.verifyAndUpdateWalletAddress(userId, web3AuthDto));
}
```

This is a separate flow from the public wallet login above, `POST /web3-auth/verify` requires an existing, valid access token (`AccessTokenGuard`), and its job is letting an already authenticated user attach a wallet address to their account rather than log in with one. It follows the same nonce plus `decodeSignature` pattern as `authenticate`, scoped to the current user's `userId` and network rather than a fresh, anonymous lookup by wallet address, and correctly rejects with `UserErrorMessage.USER_WITH_THIS_WALLET_ALREADY_EXISTS` if that wallet is already claimed by a different account, caught by checking for `PostgresErrorCode.UNIQUE_VIOLATION` the same way `AuthService.register` does for duplicate emails.

## The Solana login flow, which never checks a signature

```ts
// src/components/web3-auth/web3-auth.service.ts
async solanaLogin(walletAddress: string): Promise<NonceResponseDto> {
    const nonce = generateNonce(32);
    try {
        await this.userService.updateNonce(walletAddress, nonce);
        return new NonceResponseDto(nonce);
    } catch (err) {
        if (err instanceof HttpException && err.getStatus() === HttpStatus.NOT_FOUND) {
            await this.registerSolana(walletAddress, nonce, 'solana');
            return new NonceResponseDto(nonce);
        }
        throw new Error(err);
    }
}

async solanAuthenticate(singupSolanaAuth: SignupSolanaAuth): Promise<ReturnLoginDto> {
    const nonce = await this.userService.getNonceByWalletAddress(singupSolanaAuth.walletAddress, singupSolanaAuth.network);
    if (singupSolanaAuth.walletAddress && nonce) {
        const user = await this.userService.getByWallet(singupSolanaAuth.walletAddress);
        if (user.isDeleted) { throw new UnauthorizedException('...deleted...'); }
        if (user.isBlocked) { throw new UnauthorizedException('...blocked...'); }
        await this.userService.updateTheUserSource(user.id, "main_site");
        const response = await this.handleResponse(user.id);
        response.user = user;
        return response;
    } else {
        throw new UnauthorizedException(Web3AuthErrorMessage.INVALID_TOKEN);
    }
}
```

`GET /web3-auth/solana-login/:walletAddress` generates and stores a nonce for a Solana wallet address, exactly parallel to the EVM `login` method above it. `POST /web3-auth/solana-authenticate` is where the parallel breaks. Its request body, `SignupSolanaAuth`, is defined as only two fields:

```ts
// src/components/web3-auth/dto/signup-solana-auth.ts
export class SignupSolanaAuth {
    @IsNotEmpty() @IsString() walletAddress: string;
    @IsNotEmpty() @IsString() network: string;
}
```

There is no `signature` field on this DTO at all, and `solanAuthenticate`'s own logic never calls `decodeSignature`, `ethers.utils.verifyMessage`, or any Solana equivalent (a real check would use something like `tweetnacl`'s `nacl.sign.detached.verify` against the wallet's Solana public key). The entire authentication check is `if (singupSolanaAuth.walletAddress && nonce)`, meaning it passes as long as a wallet address is present in the request body and some nonce happens to be stored for it, a condition satisfied simply by having called `GET /web3-auth/solana-login/:walletAddress` first, which requires no proof of anything either.

Put plainly: as written, `POST /web3-auth/solana-authenticate` will log a caller into any Solana backed account for which they can supply the correct wallet address, with no cryptographic proof of ownership required at any point. Since Solana wallet addresses tied to a given account are not inherently secret, they are often visible in on chain activity, in a UI, or simply guessable by someone who already knows which wallet a target uses, this is a genuine account takeover path for any user who has ever logged in with a Solana wallet, not a theoretical one. This stands in direct contrast to the EVM path immediately above it in the very same file, which does the cryptographic check correctly, so this reads as a real, specific gap in this one function rather than a deliberate, documented design decision to trust Solana logins differently. It is the most significant single finding in this cluster and is worth raising for a fix rather than treating as background technical debt.

The two Solana "add wallet to existing account" helpers, `solanaAddWallet` and `solanaAddWalletWithCheckUserExisting`, have the same shape, they generate and store a nonce against a `userId` but their controller routes (`POST /web3-auth/solana-add-wallet` and `POST /web3-auth/check-solana-wallet-existence`) are not behind `AccessTokenGuard` at all, unlike the equivalent EVM `verify` route, and neither ever calls anything that checks a signature either. Whatever step is meant to complete that pairing (an actual signed message check) does not exist anywhere in this controller or service file.
