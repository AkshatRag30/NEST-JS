# 02. Coupons, and Everything a Discount Code Has to Check

## The entity encodes a surprising amount of business logic in its columns

```ts
// src/components/coupon/entity/coupon.entity.ts
@Entity({ name: 'tbl_coupons' })
export class CouponEntity extends BaseEntity {
    @Column({ unique: true }) couponCode: string;
    @Column() couponType: 'ALL' | 'UD' | 'ENS' | 'Arbitrum' | 'BinanceSmartChain' | /* ...combinations... */;
    @Column() couponValue: number;
    @Column({ default: true }) isCouponPercentage: boolean;
    @Column() couponUsageLimit: number;
    @Column() couponUsageCount: number;
    @Column({ type: 'bigint' }) couponExpiryDate: number;
    @Column({ type: 'bigint', nullable: true }) couponStartDate: number | null;
    @Column() couponStatus: 'ACTIVE' | 'INACTIVE';
    @Column({ type: 'text', array: true, nullable: true }) applicableTlds: string[];
    @Column({ type: 'text', nullable: true }) influncerId: string;
    @Column({ default: false }) isPerkCoupon: boolean;
}
```

`couponType` is a closed set of exact strings like `'Arbitrum-BinanceSmartChain-ENS-UD'`, not a join table, which tells you this system was built with a small, known set of domain providers in mind, a coupon restricted to more than one provider gets its own literal hyphenated type value rather than a proper many to many relation. `isCouponPercentage` decides whether `couponValue` means "knock this many dollars off" or "knock this percent off," and that boolean travels with the coupon everywhere it is used, including all the way into the sales invoice template later. `influncerId` (the typo is in the real column name) is what turns a coupon into an affiliate or influencer code, checked by a completely different code path (`findCouponByInfluencerId`) than a normal user typed code.

## Validating a coupon is really validating a whole cart against it

`CouponService.findCouponByCouponCode` does not just check a code exists and is not expired. It calls back into `CartServiceInterface.findCartByCartId` to resolve the actual cart rows a checkout is about to pay for, then extracts the provider and TLD of every item in the cart, and only then decides validity, inside `isCouponValid`:

```ts
if (isValid && couponType !== 'ALL') {
    const uniqueSortedProviders = [...new Set(cartProviders)].sort();
    const applicableProviders = couponType.split('-').sort();
    if (uniqueSortedProviders.length > applicableProviders.length) {
        isValid = false;
    } else {
        for (const provider of tldsListOfDomains) {
            if (!applicableTlds.includes(provider)) {
                isValid = false;
                break;
            }
        }
    }
}
```

This is coupon validity being decided as a function of the entire cart at once, not per line item, a coupon scoped to `'ENS-UD'` fails outright the moment the cart contains a provider outside that pair, and separately fails if any single domain's TLD is not in the coupon's own `applicableTlds` list. A frontend showing "apply coupon" has to expect a coupon that worked for one cart to legitimately stop working the moment an unrelated item gets added.

## Perk coupons are a second, stricter gate bolted on top

Some coupons (`isPerkCoupon: true`) come from a completely separate reputation and loyalty system (`reputation-gm-perk`, its own cluster) rather than being ordinary marketing codes. Before any of the cart based checks above even run, `findCouponByCouponCode` walks through a longer chain of eligibility checks specific to these: the coupon must have an active, date bounded `PerkCouponEntity` configuration, the calling user must meet a minimum reputation score and a derived tier (`BRONZE` through `PLATINUM`, computed locally by `deriveTier` rather than importing the real scoring module, a small deliberate duplication called out in the code's own comment to avoid a cross module import), the user must have already explicitly claimed the perk before trying to use it at checkout, and the coupon must never have been used by that user before, checked directly against `DomainOrderEntity` by matching `promoCodeUsed` and `userId`. Every one of those failures throws a distinct `ForbiddenException` message, which is exactly the kind of detail a frontend error toast should surface as written rather than replacing with a generic "coupon invalid."

## The usage counter, and a gap worth noticing

`updateCouponUsageCount` is a single, un-transacted `UPDATE ... SET "couponUsageCount" = "couponUsageCount" + 1`, called by the webhook handlers in the top level `webhooks` cluster once an order actually completes, not by anything inside this coupon module itself. That increment being a raw SQL expression rather than a read-then-write is good, it is safe against two concurrent completions racing each other on the count itself. What is not protected is the pairing between "the order actually completed" and "the usage count got incremented," those are two separate statements in two different modules with no shared transaction, so a crash between them could in principle let a coupon's real world usage run ahead of or behind its recorded count. For a percentage discount code this is a minor bookkeeping drift; it is worth remembering when reading the invoice cluster notes, because that same `promoCodeUsed` and `promoApplied` pairing on the order is also what the sales invoice template later reads to decide whether to print a discount line at all.

## Frontend note

A coupon input box on a checkout page is really calling an endpoint that revalidates the entire cart, not just the code, so the same code can flip from valid to invalid purely because the cart contents changed underneath it, and a perk coupon can be rejected for reasons (reputation score, tier, "not yet claimed") that have nothing to do with the code string itself, all of which are real, distinct, user facing states worth designing for individually rather than collapsing into one generic "invalid coupon" message.
