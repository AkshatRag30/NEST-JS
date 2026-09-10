# Cluster 01: Auth and Identity

This cluster covers every way a person or a wallet becomes an authenticated user in the Endless Domains backend, and everything that happens afterward to keep that session alive or to gate a route behind a role. It is built entirely from reading the real source under `src/components/auth`, `src/components/user-rbac`, `src/components/web3-auth`, `src/components/twitter-auth`, `src/components/google-auth`, `src/components/email-verification`, and the shared guard and password validation utilities under `src/@core`. Nothing here is inferred from naming alone, every claim is tied back to a specific file and, where useful, a real quoted snippet.

If you are coming from frontend work, the short version is this: this codebase has five distinct front doors into the same user table (email and password, Google, an EVM wallet signature, a Solana wallet address, and a separate admin login with 2FA), and all five of them end at the same place, a pair of JWTs, an access token and a refresh token, handed back to whatever called them. Everything else in these notes is either how one of those five doors works, or how the access token gets checked and refreshed afterward.

## Reading order

[01-password-login-and-jwt-tokens.md](01-password-login-and-jwt-tokens.md) is the foundation, read it first. It covers `AuthController` and `AuthService`, the register and login flow, where `bcrypt` and `argon2` each get used and why they are different, exactly where `JWT_ACCESS_TOKEN_SECRET` and `JWT_REFRESH_TOKEN_SECRET` are read from, the access token and refresh token Passport strategies, and the cookie parser connection from `main.ts` mentioned in the architecture note.

[02-oauth-google-and-twitter.md](02-oauth-google-and-twitter.md) covers the two social login components. Google is a real, working, second front door into the same user table. Twitter looks like one but, checked directly against `app.module.ts`, is not wired into the running application at all, and its service file has a hardcoded API credential worth flagging on its own.

[03-web3-wallet-auth.md](03-web3-wallet-auth.md) covers wallet based login for both EVM chains and Solana, the nonce and signature scheme this app uses to prove wallet ownership, and two real, distinct security observations: the EVM path never rotates its nonce after a successful login, and the Solana path does not check a signature at all.

[04-email-verification.md](04-email-verification.md) covers the standalone `EmailVerificationModule`, its own JWT secret and expiration, and the two different verification links (one during registration, one that can be triggered post login) that both land on the same controller.

[05-rbac-and-guards.md](05-rbac-and-guards.md) covers `user-rbac`, the internal admin login with TOTP based two factor authentication, the shape of its role and page access model versus a simple hardcoded role check, the full family of guards in `src/@core/common/guards` (access token, refresh token, admin token, super admin, marketing access, and the two IP based rate limiters), the never wired up `RecaptchaGuard`, and the shared password validation regex.

[06-frontend-to-fullstack-bridge.md](06-frontend-to-fullstack-bridge.md) is short and specifically written for someone moving from frontend to fullstack. It maps what you already know (calling a login endpoint, storing a token, redirecting after OAuth) onto what is actually new here (how the server side of token issuance, refresh, and verification is implemented), using this exact codebase as the example.

## The headline findings, if you only read one paragraph

The most serious thing found in this slice is in `web3-auth.service.ts`: the Solana login path (`solanAuthenticate`) never calls anything resembling `decodeSignature`, it authenticates a wallet purely by checking that a previously issued nonce exists for that wallet address, meaning no cryptographic proof of wallet ownership is required at all for a Solana login, unlike the EVM path a few methods above it which does verify a real `ethers.utils.verifyMessage` signature. A close second is a hardcoded Twitter API consumer key and secret sitting in plain text in `twitter-auth-service.ts`, though that entire module is not imported anywhere in `app.module.ts` and is not part of the running application today. Both are written up in full, with exact file paths and quoted code, in their respective notes below.
