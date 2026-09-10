# 01. The User Entity and Account Lifecycle

## Where this fits

The auth cluster's notes cover how a `userId` gets minted and handed back in a token. This note picks up from there, it's about what that `userId` actually points to, `tbl_user`, and what the `user` module lets an already-authenticated caller do with it.

## Every column on `User`, read as what it's actually for

```ts
// src/components/user/entity/user.entity.ts
@Entity({ name: 'tbl_user' })
@Index('idx_user_email', ['email'])
export class User extends BaseEntity {
    @Column({ unique: true, nullable: true })
    public email: string;

    @Column({ nullable: true })
    public phoneNumber?: string;

    @Column({ nullable: true })
    public name: string;

    @Column({ nullable: true })
    @Exclude()
    public password?: string;
    ...
```

`BaseEntity` (`src/@core/common/entity/base.entity.ts`) already gives every entity in this codebase a UUID `id`, a `createdDateTime`, a `lastChangedDateTime`, and, worth noticing on its own, an `isDeleted` boolean. `User` re-declares its own `isDeleted` column further down the file anyway, which means this entity effectively defines the same soft-delete flag twice, once inherited and once redundantly redeclared. It's harmless, TypeORM just uses the subclass's column metadata, but it's a small, real sign that this file has been edited by more than one person over time without anyone double-checking what the base class already provided.

`email` is `unique` but `nullable`, which matters because a user created purely through a wallet login (`createWithWallet`, further down) never gets an email at all until they add one later. Postgres treats every `NULL` in a unique column as distinct from every other `NULL`, so any number of wallet-only accounts can coexist with no email, and the uniqueness constraint only actually engages once two rows both try to claim the same real address.

`password` and `currentHashedRefreshToken` both carry `@Exclude()` from `class-transformer`. That decorator is what keeps a hashed password or refresh token from ever accidentally serializing into an API response, it's a safety net sitting underneath the DTO layer, not a substitute for it, since this service in practice never returns a raw `User` entity to a controller anyway (see below).

The rest of the columns tell you what kind of account this can be and where it came from: `isRegisteredWithGoogle`, `isRegisteredWithWallet`, `isMainSiteUser`, `isMarketplceUser` (the typo is in the real column name), `isAffiliateUser`, `isWaitlistUser`, `role` (a plain string, `'user'` or `'affiliate'`, not an enum at the database level). `isEmailVerified`, `isPhoneNumberVerified`, `isTwoFactorAuthenticationEnabled`, and `twoFactorAuthenticationSecret` track verification and 2FA state. `isBlocked` and `isDeleted` are two separate kill switches, an admin block versus an account deletion, that a frontend has to treat differently since one is (presumably) reversible and the other reflects account closure. `primaryDomainId` is a `unique` pointer to the one `.og` domain a user has designated as their "main" one, and it's the load-bearing field for the entire `builder-profile` feature covered later in this cluster. `aiQueryCount` and `aiSubscriptionStatus` gate usage of an AI feature elsewhere in the app. `isFoundingMember` and `foundingMemberAwardedAt` and `lastActiveAt` are newer-looking columns tied into the reputation and GM streak system referenced from `builder-profile`.

## The relations tie this entity to almost every other cluster

```ts
@OneToMany(() => WalletAddressEntity, (wallet) => wallet.user)
public walletAddresses: WalletAddressEntity[];

@OneToOne(() => AffiliateUserEntity, (affiliate) => affiliate.user, { cascade: true })
@JoinColumn()
public affiliateUser: AffiliateUserEntity;

@OneToMany(() => AffiliateKeyRequestEntity, (request) => request.user)
public affiliateKeyRequests: AffiliateKeyRequestEntity[];

@OneToMany(() => CartEntity, (cart) => cart.user)
cart: CartEntity[];

@OneToMany(() => AffiliatePayoutRequestsEntity, (pr) => pr.user)
public payoutRequests: AffiliatePayoutRequestsEntity[];
```

`User` is a genuine hub entity. It owns its wallets, its shopping cart, and reaches into the affiliate cluster in three separate directions. This is exactly the kind of file where, in a smaller project, you'd expect one focused entity, here it's the one table nearly every other feature module ends up joining against, which is normal for a several-year-old production system but worth knowing before you go looking for "the user table" and find it wired into things that have nothing to do with identity on the surface.

## What the controller actually exposes

```ts
// src/components/user/user.controller.ts
@Put()
@UseGuards(AccessTokenGuard)
public async update(@Body() userDto: UpdateUserDto, @Req() req: Request): Promise<Response> {
    const userId = req.user['userId'];
    userDto.id = userId;
    return new Response(UserSuccessMessage.USER_UPDATED_SUCCESSFULLY, await this.userService.update(userDto));
}
```

Every route on `UserController` is `@Controller('users')`, so update lives at `PUT /api/v1/users`, not `PUT /api/v1/users/:id`. The `id` never comes from the URL or the request body a client actually sent, it's overwritten server-side from the JWT (`req.user['userId']`) right before the DTO reaches the service. That's a deliberate, worth-noticing pattern, it means there is no way to pass someone else's id in the body and edit their account, the only account this endpoint can ever touch is the one the access token belongs to.

There are, unusually, two nearly identical handlers registered on the exact same route:

```ts
@Get()
@UseGuards(AccessTokenGuard)
public async getById(@Req() req: Request): Promise<Response> { ... }

@Get()
@UseGuards(AccessTokenGuard)
public async getByToken(@Req() req: Request): Promise<Response> { ... }
```

Both are `@Get()` on `@Controller('users')`, so both map to `GET /api/v1/users`. Nest resolves route collisions by registration order, so `getById` is the one that actually runs, `getByToken` is dead code that will never execute, a leftover from a rename or a merge that nobody removed. This is worth flagging exactly the way the config note in this repo's earlier docs describes, not a bug that breaks anything today, but a real, small piece of debt.

## The update flow's real validation, not just DTO shape

`class-validator` on `UpdateUserDto` only checks that `email` looks like an email and `phoneNumber` is a string, it has no way to know whether that email or phone number is already taken by a different row. That check happens by hand, inside `UserService.update`:

```ts
// src/components/user/user.service.ts
if (userDto.email != '') {
    const userByEmail = await this.userRepo.findByEmailWithWhereNotEqualToId(userDto.id, userDto.email);
    if (userByEmail.length != 0) {
        throw new ConflictException(UserErrorMessage.USER_WITH_THIS_EMAIL_ALREADY_EXISTS);
    }
}
if (userDto.phoneNumber != '') {
    const getUserByPhoneNumber = await this.userRepo.findByPhoneNumberWithWhereNotEqualToId(userDto.id, userDto.phoneNumber);
    if (getUserByPhoneNumber.length != 0) {
        throw new ConflictException(UserErrorMessage.USER_WITH_THIS_PHONE_ALREADY_EXISTS);
    }
}
```

Both queries exclude the caller's own row (`WHERE id != :id`) before checking for a collision, otherwise every update would conflict with itself. The email side of this is more interesting than it looks, because of Gmail's dot and plus-alias rules:

```ts
// src/components/user/user.repo.ts
async findByEmailWithWhereNotEqualToId(id: string, email: string): Promise<User[]> {
    // `email` is already normalised by user.service.ts before this call.
    // The CASE expression mirrors normalizeEmail() at the DB level so that
    // legacy records stored without dot-removal (e.g. "guru.prasad@gmail.com")
    // are still caught as conflicts when the incoming normalised form
    // ("guruprasad@gmail.com") matches them semantically.
    return this.userRepository
        .createQueryBuilder('user')
        .where('user.id != :id', { id })
        .andWhere(
            `CASE
                WHEN LOWER(SPLIT_PART(user.email, '@', 2)) IN ('gmail.com', 'googlemail.com')
                THEN REPLACE(LOWER(SPLIT_PART(user.email, '@', 1)), '.', '') || '@gmail.com'
                ELSE LOWER(user.email)
             END = :email`,
            { email }
        )
        .getMany();
}
```

`normalizeEmail` (`src/@core/utils/email-normalizer.util.ts`) is the TypeScript version of this same rule, run once before it ever reaches the database, lowercase, trim, and for `gmail.com`/`googlemail.com` addresses specifically, strip every `.` from the local part and cut off anything after a `+`. `guru.prasad+work@gmail.com` and `guruprasad@gmail.com` normalize to the same string, because Gmail itself treats them as the same inbox. The SQL `CASE` expression above exists because that normalization rule wasn't always applied, older rows in the table can still have dots in their stored email, so the conflict check has to reproduce the same folding logic at the database level or it would miss a real duplicate sitting right next to it. `UserRepo.findByEmailWithoutRestrictions` documents the same gap directly in a comment, checking the normalized form first and falling back to a raw, lowercase-only match for exactly those pre-normalization legacy rows.

This is a genuinely good example of a lesson that doesn't show up in a smaller project, uniqueness isn't just a database constraint you add once, it's a rule about what counts as "the same value" that can itself change over time, and a change like that has to be reconciled against data that was written under the old rule.

## What actually gets handed back to the client

`UserService` never returns a raw `User` entity. Every public method funnels through one of three private mapping functions, `toReturnUserDto`, `toReturnUserDtoWithMultipleWalltAddress` (the typo is real), or `toReturnUserForgetPasswordDto`, each of which copies an explicit, named list of fields onto a plain DTO class. `password`, `currentHashedRefreshToken`, `twoFactorAuthenticationSecret`, and `forgot_password_verification` (outside the one DTO built specifically to carry it) simply never get copied over. This is a stronger guarantee than the entity's own `@Exclude()` decorators, since it doesn't depend on `class-transformer` correctly serializing the object on every code path, the sensitive columns are never even attached to what leaves the service.

## Two account-creation paths that already reach into wallet-address

```ts
async create(userDto: CreateUserDto): Promise<ReturnUserDto> {
    const user = new User();
    user.email = normalizeEmail(userDto.email);
    user.password = userDto.password;
    ...
    const userSaved = await this.userRepo.save(user);

    const wallet = new WalletAddressEntity();
    wallet.network = 'evm_network';
    wallet.walletAddress = null;
    wallet.nonce = null;
    wallet.userId = userSaved.id;
    await this.walletRepo.save(wallet);
    return await this.toReturnUserDto(userSaved);
}
```

Both a plain email signup and a Google signup (`createWithGoogle`) immediately create an empty placeholder wallet row alongside the user, network `'evm_network'`, address `null`. That detail only makes sense once you've read the `wallet-address` note next, it's the reason every user, even one who has never touched a crypto wallet, already has a row in `tbl_wallet_address` the moment their account exists.
