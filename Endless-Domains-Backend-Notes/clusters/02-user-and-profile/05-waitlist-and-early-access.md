# 05. Waitlist and Early Access

## The headline finding first: registration is switched off

Before anything else about how this module works, the single most important thing to know is that its main entry point currently does nothing but reject every request:

```ts
// src/components/waitlist/waitlist.service.ts
async register(dto: CreateWaitlistDto, ip: string): Promise<any> {
    throw new ServiceUnavailableException('Waitlist registration is currently closed.');
    // const walletAddress = dto.walletAddress;
    // const email = normalizeEmail(dto.email);
    // ... roughly seventy more lines of real, complete, working-looking logic ...
}
```

`POST /waitlist`, the endpoint a public signup form would actually call, is wired all the way through, guarded by nothing (no auth required, by design, since a waitlist signup happens before anyone has an account), and immediately throws a 503 before it does anything else. Everything below the throw is commented out, not deleted, a full implementation: normalize the email, check for a wallet-address collision, check for an email-tied wallet mismatch, create the `User` row inside a database transaction if one doesn't already exist, generate a unique referral code, create the `WaitlistEntity` node, process a referral if one was supplied, and emit a `WAITLIST_REGISTERED` event that presumably triggers a welcome email elsewhere in the app.

This is worth reading in full before assuming the waitlist doesn't work, because everything else it exposes very much does. The leaderboard, `getNodeByEmail`, referral code validation, the entire admin surface (list, filter, export CSV, freeze/unfreeze, mark fraud, top-500 lookup, send the early-access email), all of that reads live data out of `tbl_waitlist` and works exactly as written. What's switched off is specifically new signups. The most likely real-world explanation, given the surrounding code, is that this was a genuine pre-launch, early-access capture campaign that ran its course, the company presumably hit whatever cutoff it was collecting toward, and closed the gate rather than removing the machinery that got it there. It's a good example of what "temporarily disabled in production" actually looks like in a real codebase, not a feature flag, just a hard throw sitting in front of code that still fully exists.

## An entity that stores no email

```ts
// src/components/waitlist/entity/waitlist.entity.ts
@Entity({ name: 'tbl_waitlist' })
export class WaitlistEntity extends BaseEntity {
    @ManyToOne(() => User, { onDelete: 'CASCADE', eager: false })
    @JoinColumn({ name: 'userId' })
    user: User;

    @Column({ nullable: false })
    userId: string;

    @Column({ unique: true, nullable: false })
    walletAddress: string;

    @Column({ length: 20, unique: true, nullable: false })
    referralCode: string;

    @Column({ type: 'int', default: 50 })
    points: number;

    @Column({ type: 'int', default: 0 })
    referralCount: number;

    @Column({ length: 20, nullable: true })
    referredByCode: string;

    @Column({ type: 'boolean', default: false })
    isFraud: boolean;
}
```

Notice there's no `email` column here at all, despite `CreateWaitlistDto` requiring one at signup time. Every email an admin sees anywhere in this feature (the admin list, the CSV export, the top-500 lookup) comes from joining out to `tbl_user.email` through `userId`, never from a column on this table itself. `getNodeByEmail`, the public "check your waitlist status" endpoint, actually queries through that join too:

```ts
// src/components/waitlist/waitlist-repo.service.ts
async findByEmail(email: string): Promise<WaitlistEntity | null> {
    return this.waitlistRepo
        .createQueryBuilder('w')
        .innerJoin('tbl_user', 'u', 'u.id = w."userId"')
        .where('u.email = :email', { email })
        .getOne();
}
```

So the waitlist row and the account it points to are two genuinely separate concepts, joined by `userId`, exactly the same way `builder-profile` treats its own domain ownership as separate from the account. Every registration, had it still been active, would have to first find-or-create a `User` row (`isWaitlistUser: true`), and only then create the waitlist node pointing at it, this table never independently owns the email that was used to sign up.

Every wallet-facing string is stored twice, once as the real address, `walletAddress`, marked `unique`, and once as a display-friendly, truncated version, `walletAddressDisplay` (`0x1234...abcd` style), computed once at registration time and never recomputed. Points start every new signup at fifty, and successful referrals add a hundred more to the referrer, both hardcoded constants inside the (currently unreachable) registration logic rather than configuration.

## The leaderboard's rank math

The leaderboard isn't a stored rank column, it's computed live, per row, with a correlated subquery counting how many other rows outrank this one:

```ts
// src/components/waitlist/waitlist-repo.service.ts
const { entities, raw } = await this.waitlistRepo
    .createQueryBuilder('w')
    .leftJoinAndSelect('w.user', 'u')
    .addSelect(
        (qb) => qb
            .select('COUNT(*)')
            .from(WaitlistEntity, 'w2')
            .where('w2.points > w.points OR (w2.points = w.points AND w2."createdDateTime" < w."createdDateTime")')
            .andWhere('w2."isFraud" = false'),
        'nodes_ahead',
    )
    .where('w.isFraud = false')
    .orderBy('w.points', 'DESC')
    .addOrderBy('w.createdDateTime', 'ASC')
    .skip(skip)
    .take(limit)
    .getRawAndEntities();

const data = entities.map((entity, i) => ({ ...entity, computedRank: parseInt(raw[i].nodes_ahead, 10) + 1 }));
```

Rank is "how many other non-fraud rows have strictly more points, or the same points but an earlier signup time, plus one." That tiebreaker matters, without it, two people tied on points would both claim the same rank rather than the earlier signup winning it. `isFraud` rows are excluded from the count entirely, not just hidden from the list, so a fraudulent entry doesn't even artificially push everyone below it down a spot. `getNodeRank`, used by the single-user lookup endpoints, runs the exact same comparison as a plain count rather than a join, computing one person's rank in isolation instead of a whole page of them.

## Referral abuse prevention, capped per IP per referrer per day

The (currently dormant) `processReferralInTx` method is worth reading for the pattern even though it can't run right now, because the anti-abuse logic it contains is real and reusable:

```ts
const limit = Number(this.configService.get<string>('WAITLIST_REFERRAL_IP_LIMIT') || REFERRAL_IP_LIMIT_DEFAULT);
const since = new Date(Date.now() - 24 * 60 * 60 * 1000);
const ipReferralCount = await manager
    .createQueryBuilder(WaitlistReferralEntity, 'r')
    .where('r."referrerCode" = :referrerCode', { referrerCode: referredByCode })
    .andWhere('r."referrerIp" = :ip', { ip })
    .andWhere('r."createdDateTime" > :since', { since })
    .getCount();

if (ipReferralCount >= limit) {
    this.logger.warn(`IP abuse cap hit — referrerCode=${referredByCode} ip=${ip}`);
    return;
}
```

Every successful referral gets logged into its own table, `WaitlistReferralEntity`, carrying the referrer's code, the new signup's wallet address, and the IP address the new signup came from. Before crediting a referral, the code counts how many referrals that same referrer's code has picked up from that same IP in the last 24 hours, and simply stops crediting (returns silently, no error surfaced to the referred user) once a configurable ceiling is hit, meaning one person can't sit behind a single IP and farm referral points onto their own code by repeatedly "referring" themselves. `WaitlistReferralEntity` backs this with its own composite constraints:

```ts
// src/components/waitlist/entity/waitlist-referral.entity.ts
@Index('idx_waitlist_referral_abuse', ['referrerCode', 'referrerIp', 'createdDateTime'])
@Unique(['referrerCode', 'referredWalletAddress'])
export class WaitlistReferralEntity extends BaseEntity { ... }
```

the `Unique` pair stops the exact same wallet from being credited as a referral under the same code twice, and the index exists purely to make the abuse-window query above fast. `AdminWaitlistController`'s `GET /admin/waitlist/flagged` surfaces exactly this same shape from the other side, admin-facing, grouping referral records by referrer and IP and returning any pair that's crossed the threshold, for a human to review manually.

## The admin toolkit

`AdminWaitlistController`, gated by `AccessTokenGuard` and `SuperAdminAccessGuard`, is a genuinely complete internal ops surface for a growth campaign: paginated, filterable listing (`adminGetAll`), a CSV export that specifically re-joins the user relation so email is present in the file (`adminGetExportCsv`), freezing and unfreezing the public leaderboard (backed by a tiny generic key-value table, `WaitlistConfigEntity`, read through `getConfig('leaderboard_frozen')`), bulk fraud marking by email list (`mark-fraud`, capped at a thousand emails per call by `@ArrayMaxSize(1000)` on the DTO), pulling the top 500 eligible signups, and sending the early-access announcement email to a manually supplied list (capped at 500 by `@ArrayMaxSize(500)`, matching the "first 500" framing of the feature itself). `Top500Service.isTop500`, exposed publicly as `GET /waitlist/top500/verify` behind normal user auth, runs a `ROW_NUMBER() OVER (...)` window function directly in raw SQL to answer one boolean question, is this authenticated user inside the top 500 non-fraud rows by the same points-then-signup-time ordering used everywhere else in this feature.
