# 01. Affiliate Keys and Referral Tracking

## What an affiliate key actually is

The unit this whole referral program is built around is not a person, it is a row in `affiliate_keys`, the `AffiliateKeyEntity` defined in `src/components/affilate/key-managment/entity/affiliate-key.entity.ts`.

```ts
@Entity({ name: 'affiliate_keys' })
export class AffiliateKeyEntity extends BaseEntity {
  @Column({ unique: true })
  affiliateKey: string;

  @Column({ default: true })
  status: boolean;

  @Column({ type: 'float', default: 0, nullable: true })
  discountPercentage?: number;

  @Column({ type: 'float', default: 0, nullable: true })
  commissionPercentage?: number;

  @Column({ type: 'bigint', nullable: false })
  expiryDate: number;

  @Column({ unique: false, nullable: true })
  email: string;

  @Column({ type: 'boolean', default: false })
  public isDeleted: boolean;
}
```

That string, `affiliateKey`, is the actual referral code. It gets embedded into a URL on the frontend, `AffiliateService.createAffiliateKey` builds the shareable link as `${WEB_CLIENT_URL}${newKey.affiliateKey}` when it emails the new key to whoever it belongs to. Two numbers on the row do the real financial work, `discountPercentage` is how much the referred customer saves, and `commissionPercentage` is how much the affiliate earns, and they are independent of each other, the key can discount a purchase by one amount while paying the affiliate a completely different percentage of it. `expiryDate` is stored as a raw Unix timestamp in seconds rather than a real `Date` column, which shows up later as a small but real trap, `AffiliateService.findByAffiliateKey` has to multiply it by 1000 before comparing it against `Date.now()`.

Every affiliate key can also be tied to a `coupon`, a separate `CouponEntity` looked up by `couponService.fetchCouponByAffiliateKey(key.affiliateKey)`. This is worth noticing early because it means an affiliate key is not the discount mechanism by itself, it is a pointer to a coupon that carries its own usage limits, TLD restrictions, and status, the affiliate key and its linked coupon travel together but live in two different tables owned by two different components.

## Who requests a key, and who approves it

An affiliate does not create their own key. There is a separate, lightweight request entity, `AffiliateKeyRequestEntity` (`tbl_affiliate_key_request`), that exists purely so a user can ask for one and an admin can approve or reject that ask.

```ts
@Entity({ name: 'tbl_affiliate_key_request' })
export class AffiliateKeyRequestEntity extends BaseEntity {
    @Column({ nullable: false })
    public keyName: string;

    @Column({ type: 'text', nullable: true })
    public description: string;

    @Column({ default: 'pending' })
    public status: string;

    @ManyToOne(() => User, (user) => user.affiliateKeyRequests, { onDelete: 'CASCADE' })
    @JoinColumn({ name: 'user_id' })
    public user: User;

    @Column()
    public user_id: string;
}
```

`AffiliateUserService.createAffiliateKeyRequest` (in the sibling `affiliate-user` component, covered in the next note) is what a logged in user actually calls, and it refuses outright unless `user.isAffiliateUser` is already true, so requesting a key is a privilege that has to be granted first, not something any signed up account can do. Once a request lands, it fires an email to two hardcoded addresses, `ankit@endlessdomains.io` and `affiliate@endlessdomains.io`, which is a very plain, very direct sign that key approval today is a manual, human step rather than an automated one. `AffiliateRepo.updateAffiliateKeyRequestStatus` is what an admin endpoint eventually calls to move the request's `status` from `pending` to `approved` or `rejected`, but nothing in this code path automatically turns an approved request into a real `AffiliateKeyEntity`, that still happens through the separate `createAffiliateKey` call, so the request and the key it produces are two distinct writes an admin has to connect themselves.

## Tracking a click and a conversion

The actual attribution mechanism lives in `affilate/user-activity-managment`, and it is deliberately simple, one table, `affilate_user_activities` (`AffilateUserActivityEntity`), that stores an affiliate key, an IP address, a free text `action` string, and a JSON `payload`.

```ts
@Entity({ name: 'affilate_user_activities' })
export class AffilateUserActivityEntity extends BaseEntity {
  @Column()
  affiliateKey: string;

  @Column()
  ipAddress: string;

  @Column()
  action: string;

  @Column('simple-json')
  payload: Record<string, any>;
}
```

The frontend is the one deciding what counts as an activity. Every time someone lands on a referral link, adds a domain to a cart, opens the cart, starts checkout, or completes a payment, the frontend fires a `POST /activity` with the same `affiliateKey`, a fixed `action` string like `"Domain Search suggestion"` or `"Payment Success"`, and whatever it wants to put in `payload`. `AffilateUserActivityService.logAffiliateActivity` validates the key is real and not expired before saving, and treats the very first activity ever logged for a given `affiliateKey` and `ipAddress` pair specially, it is written with `action: 'user_added'` regardless of what the caller actually sent, which is the row `countReferredUsersByAffiliateKey` later counts to answer "how many distinct people has this link brought in".

Turning that stream of loosely typed rows into money is the interesting part. `InfluencerActivityRepo.getTrackedActionsByAffiliateKey` runs a raw SQL query against `affilate_user_activities`, grouping by `action`, but for `Payment Success` rows specifically it does something unusual, it extracts an order id out of the payload text with a Postgres `SUBSTRING` pattern match and counts distinct order ids instead of counting rows:

```sql
CASE 
  WHEN "action" = 'Payment Success' THEN COUNT(DISTINCT
    SUBSTRING("payload" FROM 'Order ID: ([^,]+)')
  )
  ELSE COUNT(*)
END AS "actionCount"
```

`getTotalSalesAndCommissionByAffiliateKey` goes a step further, it pulls every `Payment Success` row for the key, regex matches `Order ID: ([\w-]+)` out of each payload's `message` field in application code, then runs a second query against `tbl_domain_order` for exactly those ids where `orderStatus = 'Completed'`, sums `totalCost`, and multiplies by the key's `commissionPercentage`. In other words, there is no dedicated event or webhook that tells this system "a sale happened because of affiliate X", the entire commission calculation is reconstructed after the fact by parsing an order id back out of a human readable activity log message, then cross checking that order actually exists and actually completed. It works, but it means the exact wording of that payload message, `Order ID: ...`, is load bearing, and a frontend change to that string would silently break commission totals without throwing any error.

## Reports, and a download link that is itself the auth

`AffilateUserActivityController` and its service also generate spreadsheet reports, either a full report across all of an affiliate's keys (`generateAffiliateReport`, emailed) or a single key report (`generateAffiliateReportForSingle`, downloaded). The single report path is worth a look because of how it handles authorization on the download itself, the report is uploaded privately to S3, and instead of protecting the download endpoint with `AccessTokenGuard`, the service signs a short lived JWT scoped to that one S3 key and returns a `downloadUrl` containing the token:

```ts
const token = this.jwtService.sign(
  { s3Key, purpose: 'affiliate-report-download' },
  { secret: this.JWT_SECRET, expiresIn: '1d' }
);
```

`GET /activity/report/download` has no `@UseGuards` at all, `downloadAffiliateReport` just verifies the token, checks its `purpose` field matches, and streams the file back. The token itself is the credential, valid for one day, scoped to exactly one file, which is a reasonable pattern for a one off download link but is worth recognizing as different from every other admin route in this cluster, which gate access with `SuperAdminAccessGuard` or `AccessTokenGuard` instead.
