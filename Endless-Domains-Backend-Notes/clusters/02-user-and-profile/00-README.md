# 02. User and Profile

## What this cluster covers

This cluster is about what a user actually is once the auth cluster's login flow has already done its job and handed back a `userId`. Four components live here, `user`, `wallet-address`, `builder-profile`, and `waitlist`, and they answer four different questions.

`user` is the account record itself, the row in `tbl_user` that every other cluster in this codebase points back to, one way or another. `wallet-address` is a small, controller-less data module that stores the blockchain wallets linked to an account, and is the seam where this cluster hands off to the Web3 auth and blockchain integration clusters covered elsewhere. `builder-profile` is a genuinely public-facing feature, a personal, link-in-bio style page tied to a user's primary `.og` domain, complete with avatar upload, projects, socials, and even a read-only rollup of that domain's reputation score and NFT achievements. `waitlist` is a pre-launch, early-access signup system with its own leaderboard and referral logic, and reading it in full turns up something worth knowing before you go looking for it elsewhere, its main registration endpoint is currently switched off in production.

## Files in this folder

[01-user-entity-and-account-lifecycle.md](01-user-entity-and-account-lifecycle.md) walks through every column on `User`, what `UserService` actually does with them, and the update flow's email and phone uniqueness checks.

[02-wallet-address-and-web3-linkage.md](02-wallet-address-and-web3-linkage.md) covers `WalletAddressEntity`, why a brand-new user already owns a wallet row with a null address, and how this table is the join point for the wallet-login flow covered in the auth cluster.

[03-builder-profile-link-in-bio.md](03-builder-profile-link-in-bio.md) covers the public builder profile feature end to end, creation, the avatar upload pipeline, projects, socials, and the public read endpoint that stitches in reputation and NFT data from three other clusters.

[04-builder-profile-ownership-and-admin.md](04-builder-profile-ownership-and-admin.md) covers the one piece of `builder-profile` that is genuinely subtle, the domain-ownership staleness check that guards every write and every public read, plus the full admin side (list, search, archive, restore, stats).

[05-waitlist-and-early-access.md](05-waitlist-and-early-access.md) covers the waitlist and referral system, its leaderboard ranking math, its admin fraud-marking and CSV export tools, and the fact that registration itself currently throws a 503 by design.

## What's familiar here versus what's genuinely new

If you've built profile pages, signup forms, and avatar uploads on the frontend before, a lot of the shape here will feel familiar. A `PUT /profile` that patches only the fields you send, a `POST /profile/avatar` that takes a multipart file and gives back a URL, a public `GET /profile/:domain` that renders someone else's page, none of that is a new idea, you've consumed APIs shaped exactly like this from the other side.

What's new is everything the backend is doing to make sure that data means what it claims to mean once it's actually stored. A frontend form can disable a username field or grey out a button, but it can't stop two different accounts from claiming the same email, that's why `UserService.update` runs a real database query, `findByEmailWithWhereNotEqualToId`, before it lets an email change through, and that query itself has to account for the fact that `guru.prasad@gmail.com` and `guruprasad@gmail.com` are the same inbox to Gmail even though they're different strings in a column. A frontend can hide the "publish" toggle behind a domain-ownership check performed once when the page loads, but `builder-profile` re-checks that ownership, with a 60 second cache, on every single write and every single public read, specifically because a `.og` domain is an NFT that can be sold out from under its owner between one request and the next, and a page that used to render someone's real profile has to 404 immediately once that happens, not eventually. That gap, between a client trusting its last known state and a server that has to assume that state might already be wrong, is most of what's actually being taught in this cluster.
