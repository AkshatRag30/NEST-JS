# 07. Security and Quality Findings, Collected in One Place

## Why this file exists

Eleven separate agents each read one slice of this codebase in isolation, and several of them found something genuinely worth a second look, a real gap, a piece of dead code, or an inconsistency between two similar features. Each finding is already written up in full, with real quoted code, inside its own cluster's notes. This file exists purely to collect them in one place, ranked roughly by how serious they are, so nothing gets lost across eleven different folders, and so you have one list to actually act on rather than eleven.

A note on tone before the list itself. Every one of these was found by reading real, live, production code for a real company, not a toy project, and every large, multi year codebase built by many people accumulates issues like this, that is normal, not a verdict on anyone's competence. The skill worth taking from this file is not judgment, it is the habit of noticing, and knowing what to do once you notice, which for anything below marked genuinely security relevant is simply this, tell whoever owns that part of the system, do not fix it quietly on your own without discussion, since a change to auth or payment code always needs a second set of eyes before it ships.

## Findings serious enough to raise with the team directly

The Solana wallet login path in `web3-auth.service.ts` never checks a cryptographic signature at all, unlike the Ethereum path, which correctly uses `ethers.utils.verifyMessage`. Anyone who knows a Solana address already tied to an existing account can log in as that user with no proof of ownership. Full detail in [clusters/01-auth-and-identity/03-web3-wallet-auth.md](clusters/01-auth-and-identity/03-web3-wallet-auth.md). This is a real authentication bypass.

The Unstoppable Domains webhook handler computes an HMAC signature and then never actually checks it, the comparison line is commented out, and the secret it computes that unused signature with is a literal string hardcoded in the source file rather than pulled from Secrets Manager the way every other secret in this app is handled. The main Stripe webhook, which confirms real domain purchase payments, performs no signature verification of any kind. Full detail in [clusters/09-events-and-webhooks/02-signature-verification-deep-dive.md](clusters/09-events-and-webhooks/02-signature-verification-deep-dive.md). A public webhook endpoint with no real verification can be called by anyone, not just the provider it claims to trust.

Several admin write endpoints, across FAQs, testimonials, static content, and a few search management routes, carry no authentication guard at all, sitting directly beside sibling endpoints in the very same controllers that are properly locked down. Full detail in [clusters/07-admin-cms-and-marketing/02-static-content-and-tld-metadata.md](clusters/07-admin-cms-and-marketing/02-static-content-and-tld-metadata.md) and [08-internal-search-analytics-and-event-tracking.md](clusters/07-admin-cms-and-marketing/08-internal-search-analytics-and-event-tracking.md).

The admin facing user management response DTOs in `user-rbac` return a user's password directly in the response body, the plaintext value at creation time, the bcrypt hash on login, while the ordinary consumer facing user DTO correctly has no password field at all. Full detail in [clusters/01-auth-and-identity/05-rbac-and-guards.md](clusters/01-auth-and-identity/05-rbac-and-guards.md).

## Real, worth fixing, but lower stakes

In the buy domain flow, reserving a listing and saving the actual purchase record happen as two separate, un-transacted writes. If the second write fails, the listing is left stuck "in progress" permanently with no purchase behind it, and nothing ever automatically revisits it. Full detail in [clusters/03-commerce-and-marketplace/03-domain-listing-and-buy-domain-secondary-marketplace.md](clusters/03-commerce-and-marketplace/03-domain-listing-and-buy-domain-secondary-marketplace.md). Across the whole commerce cluster, this is the only place a payment or order state change is genuinely vulnerable to a partial failure leaving bad data behind.

A Twitter OAuth consumer key and secret sit hardcoded in plain text in `twitter-auth-service.ts`. This is a real, leaked credential sitting in version control, even though the module itself is never actually imported into the running app today. Full detail in [clusters/01-auth-and-identity/02-oauth-google-and-twitter.md](clusters/01-auth-and-identity/02-oauth-google-and-twitter.md). A credential that is not currently reachable is still a credential that should be rotated and removed, since source history keeps it around indefinitely.

## Dead or dormant code worth knowing about, not urgent

`listener`, despite its name, is not a blockchain event subscriber, it is a generic cron job registry that is injected into `domain-order.service.ts` but never actually called anywhere in the codebase. [clusters/05-blockchain-infrastructure/09-listener-what-it-really-is.md](clusters/05-blockchain-infrastructure/09-listener-what-it-really-is.md).

The waitlist signup endpoint throws a fixed `ServiceUnavailableException` before running any of its own, fully built referral and abuse prevention logic, which sits commented out directly beneath the throw. This reads as a deliberate, temporary shutoff rather than an accident. [clusters/02-user-and-profile/05-waitlist-and-early-access.md](clusters/02-user-and-profile/05-waitlist-and-early-access.md).

The recaptcha guard's success path never actually returns `true`, but the guard is also never applied to any real route, so it is broken without currently being harmful. [clusters/01-auth-and-identity/05-rbac-and-guards.md](clusters/01-auth-and-identity/05-rbac-and-guards.md).

An `arb-integration` file contains a broken, fire and forget duplicate of a working call that already exists correctly in `ens-arb-bnb-domain-suggestion`, and a `bnb-integration` module exists but is empty. [clusters/06-chain-and-registrar-integrations/06-quirks-and-bugs-catalog.md](clusters/06-chain-and-registrar-integrations/06-quirks-and-bugs-catalog.md).

## Naming and organizational history worth understanding, not bugs at all

`affilate` (the misspelled folder) and `affiliate-user` (the correctly spelled one) are not a deliberate architectural split, the older `affilate` layer owns the real referral keys and activity tracking, the newer `affiliate-user` layer turned "being an affiliate" into a real account with payouts, and the two depend on each other in both directions, down to colliding on the same dependency injection token string in two unrelated modules. [clusters/08-affiliate-and-loyalty/03-affilate-vs-affiliate-user-the-split.md](clusters/08-affiliate-and-loyalty/03-affilate-vs-affiliate-user-the-split.md).

The internal, application wide event system built on `EventEmitter2` and a completely unrelated marketing CMS module that happens to also be named `events` are two different things that share a name by coincidence. [clusters/09-events-and-webhooks/03-internal-events-vs-external-webhooks.md](clusters/09-events-and-webhooks/03-internal-events-vs-external-webhooks.md).

`ud-billing` is not a payment gateway despite sitting among the payment integration folders, it is a super admin only analytics and revenue reconciliation dashboard reading data the real payment integrations already wrote. [clusters/11-payment-gateway-integrations/04-ud-billing.md](clusters/11-payment-gateway-integrations/04-ud-billing.md).

## The pattern worth taking away from this whole list

Notice how many of these findings are not "the code is wrong," but "the code is inconsistent with itself," one wallet type checks a signature and the other does not, one webhook verifies and another does not, one DTO leaks a password and its sibling does not, one folder pair looks like a planned split but is actually an accident of history. That inconsistency, not raw complexity, is the real signature of a large, real, multi year codebase built by many hands, and learning to hunt for exactly that kind of asymmetry, the same operation done two different ways in two similar looking places, is one of the highest leverage code reading skills you can build heading into a fullstack or backend role.
