# 01. Admin User Management

## What actually lives under `admin`

The task description guessed, based on the folder name, that `admin` would hold user management, and it turned out to be exactly right and nothing more. The entire `src/components/admin` folder contains one sub-folder, `user-management`, and nothing else. Whatever other admin surfaces this company has (managing landing page content, TLD metadata, ads, and so on, all covered elsewhere in this cluster) live in their own top level component folders rather than under `admin`, so `admin` specifically means one thing here, an internal dashboard for managing the platform's end users.

## The shape of the feature

`UserManagementController` answers at `/admin/user-management` and exposes viewing a single user's details, listing all users with a long list of filters, a thirty day registration trend, a founding members list, deleting (really, soft-deleting) a user, updating a user's email verification status, blocking a user, updating a user's email address, sending a user a one off email, exporting the current filtered list to CSV, and bulk blocking a list of user ids at once. Every route requires `AccessTokenGuard`, and three of them, the registration trend, CSV export, and bulk block, add `SuperAdminAccessGuard` on top, which is a real and sensible distinction, the routes gated further are the ones that either reveal aggregate business data or act on many users in one call.

## The search logic is genuinely more interesting than it looks

`viewAllUserList` in `UserManagementRepo` does not just run one query. It first inspects whatever the caller typed into the search box and classifies it:

```ts
private detectSearchType(search: string): 'solana' | 'evm' | 'text' {
    const isSolanaAddress = search.length >= 32 && search.length <= 44 && !search.startsWith('0x');
    const isEvmAddress = search.startsWith('0x') && search.length === 42;
    if (isSolanaAddress) return 'solana';
    if (isEvmAddress) return 'evm';
    return 'text';
}
```

If the search string looks like a wallet address, on either Solana or an EVM chain, the search runs an exact address match against `tbl_wallet_address` instead of the usual name or email `ILIKE` search, deduplicating across wallets and preferring whichever matched user record actually has an email on file. This is worth noticing because it means the admin search box, from a frontend developer's point of view, is really two different endpoints wearing one input field, and the choice of which one runs happens entirely server side based on the shape of what was typed.

## Explicit column selection exists for a real security reason, not by accident

The list query does not call `find()` or `select('*')`, it names every column explicitly:

```ts
const USER_LIST_SELECT_COLUMNS = [
    'user.id', 'user.name', 'user.email', 'user.phoneNumber', 'user.isEmailVerified',
    'user.isBlocked', 'user.isAffiliateUser', 'user.isMarketplceUser', 'user.isMainSiteUser',
    'user.isFoundingMember', 'user.foundingMemberAwardedAt', 'user.role',
    'user.isTwoFactorAuthenticationEnabled', 'user.isRegisteredWithGoogle', 'user.isRegisteredWithWallet',
    'user.createdDateTime', 'user.isDeleted', 'user.lastActiveAt',
    'wallet.id', 'wallet.network', 'wallet.walletAddress', 'wallet.createdDateTime',
];
```

A comment directly above this explains why, the full `User` entity carries the bcrypt password hash and the current refresh token, and since no `ClassSerializerInterceptor` is registered on this route, the entity's own `@Exclude()` decorators (which would normally strip those fields automatically before a response goes out) never actually run here. Without this explicit column list, the password hash would have been serialized straight into the admin dashboard's response body. This is a genuinely good defensive pattern to recognize, and also a reminder that `@Exclude()` decorators are not a magic guarantee, they only work if the specific interceptor that reads them is actually wired into that specific route.

## No caching on the dashboard style endpoints, despite the pattern existing elsewhere

`getGlobalUserCounts` runs six separate count queries plus a correlated subquery for "power users" (anyone owning ten or more domains) every single time `getAllUsers` or `exportUsersCsv` is called, and `getRegistrationTrend` runs a `GROUP BY` over the last thirty days on every request. Both are exactly the kind of slow, aggregate, dashboard style query that the root `app.module.ts`'s comment about `CacheModule` and `CacheInterceptor` describes as the intended use case. Neither route uses `CacheInterceptor`. Compare this against [06-google-analytics-integration.md](06-google-analytics-integration.md), where the equivalent dashboard endpoints do cache. This is not a bug exactly, since correctness does not depend on it, but it is a real, checkable gap between a documented intent and what one specific controller actually does.

## Value scoring and CSV export

Every user returned from the list or export endpoints gets three numbers attached by `enrichWithDomainCounts`, computed with two aggregated queries covering the whole page of results at once rather than one query per user (explicitly called out in a comment as avoiding an N+1 pattern): `totalDomainsOwned`, `totalOrders`, `totalSpend` (converted from stored cents to dollars), and a derived `userValueScore` equal to `domainCount * 2 + totalSpend * 0.5 + tenureDays * 0.1`. The CSV export endpoint (`GET /export-csv`, superadmin only) runs the exact same filtering and enrichment logic as the list endpoint and then hands the result to the `json2csv` package to build the file, so the numbers an admin sees on screen and the numbers they download are guaranteed to match, because they come from the same code path.

## Email actions go through the event system, not directly

`updateUserEmail` and `sendEmail` do not call an email provider themselves. They call `this.eventEmitter.emit(...)` with `UPDATE_EMAIL` or `SENT_EMAIL` payloads, which some other listener elsewhere in the codebase (outside this cluster) presumably picks up and turns into an actual SendGrid call. One specific line is worth quoting as a real, live oddity rather than glossing over it:

```ts
this.eventEmitter.emit(LOGSNAG_IDENTIFY, {
    user_id: 'a75dcc41-be82-4aee-ab2e-dbb94cc5aa4f',
    properties: { email: '', name: '', walletAddress: '' }
} as IdentifyOptions);
```

This fires every time an admin updates a user's email, with a hardcoded user id and every property left as an empty string. Whatever this was meant to do for LogSnag (the analytics identify service this project depends on), it is not doing it correctly today, and it reads like a debugging leftover that shipped.
