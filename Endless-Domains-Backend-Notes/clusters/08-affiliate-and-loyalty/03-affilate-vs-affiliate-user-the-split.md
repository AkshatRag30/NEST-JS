# 03. Affilate vs Affiliate User, Verified Rather Than Assumed

## The question worth answering carefully

`src/components/affilate` and `src/components/affiliate-user` sit right next to each other, one letter apart in spelling, and both have "affiliate" in nearly every class name they export. It would be easy to assume this is just a typo that got baked into a folder name and never fixed. Reading the actual entities, modules, and dependency graph tells a more specific story, one where the misspelled `affilate` folder is genuinely the older, lower level system, and `affiliate-user` was added later as a proper account layer on top of it, and the two were never merged even though the newer one depends heavily on the older one.

## What lives in each, by entity

`affilate` (missing the second "i") owns three things, and all three predate the idea of an affiliate having their own account:

`key-managment` owns `AffiliateKeyEntity` (`affiliate_keys`), the referral code itself, and `AffiliateKeyRequestEntity` (`tbl_affiliate_key_request`), the request to get one issued. `user-activity-managment` owns `AffilateUserActivityEntity` (`affilate_user_activities`), the click and conversion log described in the first note in this cluster. `affiliate-user-dashboard-managment` owns no entities of its own at all, it is pure aggregation, a service that calls into the other two plus into `affiliate-user` to build one dashboard response.

`affiliate-user` (correct spelling) owns everything that treats an affiliate as an account holder with money owed to them, `AffiliateUserEntity` (`affiliate_user_tbl`, one to one with `User`), `AffiliatePayoutRequestsEntity` (`affiliate_payout_requests_tbl`), and `AffiliateTransactionsEntity` (`affiliate_transactions_tbl`).

Nothing in `affilate` knows `affiliate-user` exists. Its modules, `AffilateModule`, `AffilateUserActivityModule`, only import each other, `LoggerModule`, `EmailVerificationModule`, and `S3Module`. The dependency runs entirely in the other direction.

## The dependency graph tells the real story

`AffiliatUserModule` (the module inside `affiliate-user`, note that class name itself drops a letter) imports `AffilateModule` directly:

```ts
// src/components/affiliate-user/affiliate-user.module.ts
imports: [TypeOrmModule.forFeature([...]), AffilateModule, EmailVerificationModule, UserModule, CouponModule],
```

And `AffiliateUserService` injects the older module's repository under its own interface token to do real work, creating a key request and fetching an affiliate's active keys both delegate straight through to it:

```ts
@Inject('AffilateRepoInterFace')
private readonly affiliateKeyRepo: AffilateRepoInterFace,
```

Going the other direction, `AffilaiteUserDahsboardModule` (again, inside `affilate`, and again its own class name is misspelled) imports `AffiliatUserModule` from the correctly spelled folder, specifically so its dashboard service can reach `AffiliateUserRepoInterface` and read or update the ledger totals on `AffiliateUserEntity`. So the dependency is not one directional across the whole system, `affiliate-user` depends on `affilate`'s key and activity tracking to know what an affiliate has earned, while `affilate`'s dashboard submodule depends back on `affiliate-user` to read and adjust that affiliate's stored totals. The two folders are genuinely entangled, just asymmetrically, one small piece of `affilate` reaches forward into the newer system, while most of the newer system reaches backward into the older one.

## Where the User entity itself settles it

The clearest evidence that these are two distinct, deliberately separate concerns, not a stray duplicate, is what `User` itself points to. It carries a one to one relation to `affiliateUser` (the `affiliate-user` folder's entity) and a one to many relation to `affiliateKeyRequests` (the `affilate` folder's entity), as two entirely separate foreign key relationships on the same row:

```ts
// src/components/user/entity/user.entity.ts (relevant relations)
public affiliateUser: AffiliateUserEntity;          // -> affiliate-user
public affiliateKeyRequests: AffiliateKeyRequestEntity[]; // -> affilate/key-managment
public isAffiliateUser: boolean;
public payoutRequests: AffiliatePayoutRequestsEntity[];   // -> affiliate-user
```

A user's account carries both relations at once because both systems are real and both are in active use, `isAffiliateUser` is the flag that gates whether someone is even allowed to request a key (checked in `affilate`'s domain), while `affiliateUser` is the record of who they are as a payee (owned by `affiliate-user`'s domain).

There is also a smaller, easy to miss piece of evidence for how organic this history actually was, worth flagging directly, `affilate/user-activity-managment` registers its own service under the DI token `'AffiliateUserServiceInterface'`, and the completely unrelated `affiliate-user` component registers its own, differently shaped service under that exact same string token. Nest scopes providers per module so this does not currently break anything, both modules keep their own instance behind their own token, but it means the same interface name resolves to two different services depending on which module's context you are reading. That is not the kind of collision a team plans for on purpose, it is the kind that happens when two features grow independently and nobody notices the token strings converged.

## So, deliberate boundary or organic accident

Based on what the code actually shows, this reads as an organic, slightly inconsistent history rather than a deliberate architectural split. The naming itself, one correctly spelled folder next to one that consistently is not, spread across class names, module names, and file names, is the strongest signal, nobody sets out to design a system with two spellings of the same word as its top level namespace. The dependency graph reinforces it, a genuinely planned split would likely draw a clean line, perhaps "key issuance and tracking" fully separate from "payout accounting" with a single shared interface between them, rather than the current shape where one small dashboard submodule inside the older folder reaches forward to read and mutate a table owned by the newer one. The most likely real history is that `affilate` (referral keys, click tracking) was built first, and when the business later needed affiliates to be real account holders who could request payouts and see a running balance, that work landed in a new, correctly spelled folder rather than being folded back into the one that already existed, and the dashboard was then built as a third piece that had to reach into both to show one coherent picture.
