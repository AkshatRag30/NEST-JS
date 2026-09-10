# 02. Affiliate User Registration, Payouts, and the Dashboard

## Becoming an affiliate is a real account, not a flag on an existing one

`affiliate-user` (standard spelling, its own top level component) is where "being an affiliate" turns into a full account record rather than just a permission. `POST /affiliateUser/register-affiliate` takes an email, name, password, a description of the applicant's reach, and a list of social media links, and `AffiliateUserService.registerAffiliateUser` does three things in sequence, it creates a brand new `User` row with `role: 'affiliate'`, it creates a linked `AffiliateUserEntity` row carrying the description and social links, and it sends a verification email. This is a separate signup path from the main product's user registration, someone can become an affiliate without ever having bought a domain.

```ts
@Entity({ name: 'affiliate_user_tbl' })
export class AffiliateUserEntity extends BaseEntity {
    @Column({ nullable: true })
    public userId: string;

    @OneToOne(() => User, (User) => User.affiliateUser, { onDelete: 'CASCADE' })
    @JoinColumn({ name: 'userId' })
    public user: User;

    @Column({ type: 'jsonb', nullable: false })
    public socialMediaLinks: { platform: string; url: string }[];

    @Column({ type: 'varchar', length: 50, nullable: true })
    public paymentMethod: 'wallet' | 'paypal';
    // ...walletAddress, walletCurrency, walletNetwork, paypalId,
    // totalAmount, totalAmountPaid, remainingAmount, requestedAmount, lastPaymentDate
}
```

This entity is a one to one extension of `User`, every affiliate has exactly one of these rows, and it is where the running financial ledger actually lives, `totalAmount` (lifetime commission earned), `totalAmountPaid`, and `remainingAmount` are maintained here, not recomputed on the fly every time someone asks. A new registration also fires an event that emails both `Ankit@endlessdomains.io` and `affiliate@endlessdomains.io` so a human knows a new applicant exists, the exact same "notify two hardcoded inboxes" pattern seen throughout this whole cluster.

## Requesting a payout

Once an affiliate has accumulated commission, `POST /affiliateUser/request-payout` is how they ask to actually be paid. The DTO enforces that a wallet payout must include a currency and network, and a PayPal payout must include a PayPal id:

```ts
if (payoutDto.paymentMethod === 'wallet') {
    if (!payoutDto.walletAddress) throw new BadRequestException('Wallet address is required');
    if (!payoutDto.walletCurrency) throw new BadRequestException('Wallet currency is required');
    if (!payoutDto.walletNetwork) throw new BadRequestException('Wallet network is required');
}
if (payoutDto.paymentMethod === 'paypal') {
    if (!payoutDto.paypalId) throw new BadRequestException('PayPal ID is required');
}
```

The request creates a new row in a third table, `AffiliatePayoutRequestsEntity` (`affiliate_payout_requests_tbl`), starting at `status: 'pending'`, and separately overwrites the payout details cached directly on the `AffiliateUserEntity` row itself, so the same wallet or PayPal information ends up written in two places, once as a permanent request record and once as "the affiliate's current preferred payout method". Like every other write in this cluster, submitting a payout request just fires an email to the two admin inboxes, there is no automated payment rail here, a human is expected to read the request and act on it manually.

## Settling a payout into a transaction

The actual, permanent record of money moving is a fourth table, `AffiliateTransactionsEntity` (`affiliate_transactions_tbl`), and only an admin can create one, through `POST /affiliateUser/settal-transaction` (the misspelling is in the real route). `AffiliateUserRepo.createTransaction` is where the ledger arithmetic happens:

```ts
affiliateUser.totalAmountPaid = Number(affiliateUser.totalAmountPaid) + Number(dto.amount);
affiliateUser.remainingAmount = Number(affiliateUser.totalAmount) - Number(affiliateUser.totalAmountPaid);
affiliateUser.lastPaymentDate = new Date();
```

If the transaction references a `payoutRequestId`, that payout request's `status` is flipped to `paid` and its `paidAt` timestamp is set in the same call. So the full lifecycle an admin walks through by hand is, an affiliate requests a payout, the admin reviews it outside this system, and once actually paid, the admin calls `settal-transaction` to both record the transaction and mark the originating request paid, updating the affiliate's running totals as a side effect of that one write. There is no code path that marks a payout paid without also creating a transaction, or vice versa, they are meant to happen together even though they are two separate inserts.

## What the dashboard actually shows

`AffiliateUserDashboardController` lives in `affilate/affiliate-user-dashboard-managment`, not in `affiliate-user`, and its job is to stitch together data that actually lives across both. `affiliateUserDashboardService.getAffiliateSummary(affiliateKey, ...)` answers "how is this one key doing" by combining three separate calls, the key's own discount and commission percentage from `AffiliateService`, click and sale totals from the activity tracking service covered in the previous note, and returns them as one flat object:

```ts
return {
    success: true,
    data: {
        affiliateKey,
        totalClicks: actions[0]?.clicks || 0,
        totalSales: totals?.totalSaleAfterDiscount || 0,
        totalCommission: totals?.commissionAmount || 0,
        discountPercentage, commissionPercentage, expiryDate,
        status: status ? 'active' : 'inactive',
    },
};
```

Because one affiliate can hold more than one key, `getAffiliateSummaryForAll(userId, ...)` loops over every key belonging to that user's email, sums clicks, sales, and commission across all of them, and, as a side effect, writes that summed commission back onto `AffiliateUserEntity.totalAmount` if it has drifted, `getAffiliateSummaryForAll` is quietly the place that keeps the ledger total in sync with the activity based commission calculation, not a separate reconciliation job. `getAffiliateActivityBreakdownOfAll` produces the chart data behind that summary, grouping the same raw action strings from the activity table into six human labeled buckets, "Domain Added to Cart", "Opened Cart View", "User Started the Checkout Process", and so on, the exact same grouping logic that appears again, nearly copy pasted, inside the Excel report generator covered in the previous note.

There is also a platform wide version of this, `getPlatformAffiliateSummary`, gated by `MarketingAccessGuard` rather than `SuperAdminAccessGuard`, worth noticing as a second, separate admin role that exists specifically for people who need visibility into affiliate performance without needing full super admin rights.
