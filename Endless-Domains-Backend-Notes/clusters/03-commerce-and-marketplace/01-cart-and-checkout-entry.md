# 01. The Cart, Where a Purchase Starts

## The entity is smaller than you would expect

```ts
// src/components/cart/entity/cart.entity.ts
@Entity({ name: 'tbl_cart' })
export class CartEntity extends BaseEntity {
    @Column({ nullable: false }) domainName: string;
    @Column({ nullable: false }) domainProvider: string;
    @Column({ type: 'decimal', precision: 15, scale: 2, default: 0 }) price: number;
    @Column({ nullable: true }) no_of_years: number;
    @Column({ default: false }) event: boolean;
    @ManyToOne(() => User, (user) => user.cart) user: User;
    @Column({ unique: true, nullable: false }) cartUinqueIdentifier: string;
    @Column({ unique: false, nullable: true }) chain: string;
}
```

A cart row is one domain name a user wants to buy, not a whole basket, the "basket" is just every `CartEntity` row that belongs to one `user`. `cartUinqueIdentifier` is worth noticing, it is built as `domainName + '~' + domainProvider + '~' + userId` (see `CartService.generateCartUniqueIdentifier`) and is marked `unique` on the column. That single string is what makes `saveIfNotExist` in `cart.repo.ts` work as a real idempotency guard, `insert().values(cart).orIgnore().execute()` silently does nothing if that identifier already exists, rather than throwing or creating a duplicate row, which is exactly what you want when a frontend accidentally fires the same "add to cart" click twice, or retries a failed request.

## Where the price actually comes from, and why it can be null

The interesting work happens in `CartService.createCartEntity`, called every time an item is added. Price is not a number the frontend sends and the backend trusts, it is computed server side, per `domainProvider`, and the logic branches hard by provider:

```ts
case DomainProvider.ENS: {
    const staticPrice = getDomainPriceObject.getDomainPrice(cart.domainName, cart.domainProvider);
    const ONE_YEAR_SECONDS = 31536000;
    const PREMIUM_THRESHOLD = 1.25;
    const label = cart.domainName.split('.')[0];
    const livePrice = await this.ensIntegrationService.getRentPriceUsd(label, ONE_YEAR_SECONDS);
    cart.price = livePrice > staticPrice * PREMIUM_THRESHOLD ? Math.round(livePrice) : staticPrice;
}
```

ENS names have a live, on chain rental price oracle. This code asks it for the true one year price, and only actually uses that live number if it is meaningfully higher (more than one quarter above) the static fallback price this codebase already knows, otherwise it just uses the static price. That threshold matters, it is a deliberate choice to treat the oracle as a signal for premium names specifically (three letter ENS names and similar can cost startlingly more than the flat rate) while not letting minor oracle noise change every quote. If the oracle call itself fails, the `catch` block falls back to the static price and logs the failure rather than blocking the add to cart entirely, a sensible degrade.

Unstoppable Domains works completely differently, and this is the branch worth reading slowly, because it is really answering "is this domain even still purchasable" as much as "what does it cost." UD domains in this system can come from two different sources, freshly available names checked live against UD's own availability API, or names Endless Domains already holds in its own wallet as pre minted inventory (`tbl_custom_domain`, owned by the custom domain component). The cart code checks the custom domain table first, and if a name is marked `Sold` there, or if the wallet check (`getDomainNamesInEndlessDomainWallet`) shows it is no longer actually sitting in the company's wallet, `cart.price` gets set to `null` and `cart.event` gets set to `true`. That `null` price is a real signal, not a bug, `CartService.create` throws `'Domain name no longer available'` the moment it sees a null price, right there in the add to cart call, before the row is even saved for a single add, and `createMany` (used by the bulk `sync` endpoint) filters null priced entries out of what it actually saves while still surfacing their names back to the caller as an error message, so a bulk cart sync can partially succeed.

## Checking availability again, right before checkout

`getUnavailableCartDataForCheckout` exists for exactly one reason, a domain can sit in someone's cart for a while, and by the time they actually go to pay, someone else may have bought it, or a UD pre minted name may have been claimed. This method re-checks every item in a user's cart against the live provider (ENS, Arbitrum, UD, or BNB, each with its own small `checkXDomainAvailability` helper) and returns only the ones that have gone stale, so the frontend's checkout page can show "these items are no longer available, please remove them" before the user tries to pay for something that will fail. `filterCartList`, used by the plain cart fetch endpoint, does the same check but goes further, it actually calls `removeBulk` on anything that has gone stale, silently pruning dead cart rows every time a user simply views their own cart.

## What is deliberately not here

There is no `CartService` method that finalizes a purchase. Checkout, in the sense of turning a set of cart rows into a paid order, is owned by `src/components/domain/domain-order` (`DomainOrderService`, outside this cluster), which injects `CartServiceInterface` specifically to call `findCartByCartId` (resolve the cart rows a checkout request references) and `removeByCartId` (clear them out once the order is placed). If you are tracing a purchase end to end starting from an "add to cart" click, the cart module's job ends the moment those cart rows are handed off, everything from "the user clicked pay" onward is the domain order flow, covered from this cluster's side in `06-order-management-and-payment-management.md`, and from the payment gateway side in the webhooks and payment integration clusters other agents are covering.

## Frontend note

If you have ever built a cart UI, the shape here should feel familiar, add, list, update quantity (here, `no_of_years`), remove one, remove several, sync a whole local cart up on login. The one thing genuinely worth taking away as a backend lesson is that price and availability are never trusted from the client, both get recomputed and re-verified server side at add time and again at checkout time, and a `null` price or a filtered out row is the backend's way of saying "this thing you're looking at may already be gone," which the frontend then has to be ready to react to gracefully rather than treat as a plain error.
