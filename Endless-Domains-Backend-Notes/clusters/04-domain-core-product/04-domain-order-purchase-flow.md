# 04. The Purchase Flow, From Cart to a Pending Order

## What `domain-order` actually owns

`DomainOrderService.create`, at the heart of `domain-order.service.ts`, is the single method that runs the moment a user finishes their cart and checks out. It is a long method, and worth walking through in the order it actually runs, because every step is a real business rule this company has encoded directly into the checkout path.

## Step one, verifying the cart, and a real, deliberate limitation

```ts
const cartIds = domainOrderDtoList.cartIds;
const duplicateCartIds = checkDuplicateCartIdInCreateDomainOrderDto(cartIds);
if (duplicateCartIds.length != 0) { throw new BadRequestException({ errorMessage: 'Duplicate Ids Found', cartIds: duplicateCartIds }); }
const cartInfoFromCartIds = await this.cartService.findCartByCartId(cartIds);
checkCartIdsExistsAccordingToUser(cartInfoFromCartIds, domainOrderDtoList.cartIds);
```

The order is built directly from cart rows fetched fresh from the database by id, not trusted from whatever the client sent about price or provider, which is the right way to build this, a client cannot simply tell the server what a domain costs. Right after that, a comment left directly in the code is worth reading verbatim, because it documents a real, current limitation rather than a finished design:

```ts
/**
 *  Delete this code later.
 *  since in the future according to plan user can buy domain name from multiple domain provider at once.
 *  But for now the user can buy domain names from one domain provider at a time.
 *  ...
 * **/
```

Cart items are split into six buckets by provider (`udDomainOrder`, `arbDomainOrder`, `ensDomainOrder`, `bnbDomainOrder`, `solDomainOrder`, `freenameOrder`), and `validateDomainInfoDtoArray` then enforces that only one of those buckets can actually be non empty, a single checkout can only buy domains from one blockchain provider at a time. If a user's cart genuinely mixed an `.eth` name with a `.crypto` name, this would reject the whole order, a real, present day constraint on this product worth knowing before you go looking for multi chain checkout in the frontend and cannot find it.

## Step two, building the order entity per provider

Once the single active provider is known, `orderDomain` dispatches to `DomainOrderReturnDomainOrderEntityInterface.returnDomainEntity`, covered in full in the next note since it is really the first half of the blockchain handoff, then this service fills in the price, user id, and, for ENS specifically, calls out to estimate a real network transaction fee and adds a twelve and a half percent surcharge on top of it before folding that into the total the customer is charged:

```ts
const estimatedFee = await this.ensIntegrationService.estimateFees(mutatedDomainEntity.domainNames, mutatedDomainEntity.durations);
const additionalTxnFee = estimatedFee * 0.125;
const totalFee = estimatedFee + additionalTxnFee;
amount = orderDomainEntity.totalCost * 100 + totalFee * 100;
```

The `* 100` throughout this method is because Stripe, like most payment processors, wants amounts in the smallest currency unit, cents, not dollars, a detail worth remembering anywhere else in this codebase you see a payment amount multiplied by one hundred.

## Step three, coupons

```ts
if (domainOrderDtoList.couponCode) {
    couponStatus = await this.couponService.findCouponByCouponCode(domainOrderDtoList.couponCode, cartIds, user.id);
    if (!couponStatus.isValid) throw new BadRequestException(couponStatus.message);
    if (couponStatus.iscouponPercentage) {
        amount = amount - (amount * couponStatus.couponValue) / 100;
        ...
    } else {
        const couponInCents = couponStatus.couponValue * 100;
        if (couponInCents >= amount) { amount = 0; ... }
        else if (amount > couponInCents) { amount = amount - couponInCents; ... }
    }
}
```

Coupons can be percentage based or a flat value, and a flat value coupon that is worth more than the order itself simply zeroes the order out rather than the customer receiving change, a sensible real world business rule. Notice the order entity itself carries `promoApplied`, `promoCodeUsed`, and `promoValue` so this discount is permanently recorded on the order row, not just applied and forgotten.

## Step four, taking payment, or skipping it entirely

```ts
if (orderDomainEntity.domainProvider !== DomainProvider.Bonfida && amount > 0) {
    getStripeClientId = await this.stripeIntegrationServiceInterface.getStripeInfoFromAPI(amount, 'USD');
    orderDomainEntity.paymentClientId = getStripeClientId.paymentClientId;
    orderDomainEntity.paymentIntentId = getStripeClientId.paymentIntentId;
    orderDomainEntity.paymentMethod = getStripeClientId.paymentMethod;
}
```

Two conditions matter here. Bonfida (Solana names) is deliberately excluded from Stripe entirely, since that flow is paid for directly in a crypto wallet transaction and settled through `updtateSolanaOrderStatus` later rather than a card. And if a coupon has already brought the amount down to exactly zero, Stripe is skipped altogether and the order is completed immediately through `udV3DomainClaimProcess`, a genuinely free order, fully paid by a coupon, never touches a payment processor at all.

## Step five, saving, and a UD specific side effect

Once the order entity is saved, if the provider is Unstoppable Domains, any domain in that order whose mint already came back `COMPLETED` (this happens for the special "event" domains covered in the next note, names this company already holds in its own wallet and is transferring rather than minting fresh) gets an additional `UDDomainOrderEntity` row created, recording a domain claim operation against Unstoppable Domains' own v3 API. This is a second, UD specific bookkeeping table sitting alongside the general purpose order and mint tables, needed because UD's own claim process has its own operation id and status vocabulary that does not map cleanly onto this company's generic order model.

Finally the cart is cleared for those ids, and any favorited domains from that same provider are automatically cleared from the user's favorites list, on the reasonable assumption that a domain you just bought is no longer something you need to keep watching.

## Cancelling and stale order cleanup

`cancelOrder` only allows cancelling an order that still belongs to the requesting user and is still in `PENDING` status, flips every domain and mint record inside it to `FAILED`, and, for anything paid through Stripe, actively cancels the payment intent rather than leaving it dangling. Separately, `updateProcessingDomainOrderOlderThen10Minutes`, called at the top of `getAllDomainOrdersByUser`, quietly resets any order that has been sitting in `PROCESSING` for more than ten minutes back to `Pending`, on the assumption that a blockchain transaction that has not confirmed in ten minutes has probably failed or been abandoned by the user's wallet, rather than leaving that order stuck forever in a state that looks like it is still happening.
