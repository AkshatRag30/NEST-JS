# Endless Domains Backend, Study Notes

## What this is

This folder is a complete set of notes on a real, live, production NestJS backend, `endless-domain-api`, belonging to a Web3 domain marketplace company called Endless Domains. The real codebase lives at `D:\Akshat Endless\backend\backend`. Every note in this folder was written by actually reading that codebase's real source files, entities, controllers, services, guards, contracts, and infrastructure configuration, in full, not by guessing from folder names or descriptions. Nothing inside the real codebase folder was ever modified, edited, or run, this entire analysis was done strictly read only, and it stays that way, this folder exists purely to hold the notes.

This exists to answer one question, how do you actually understand a large, real, unfamiliar codebase, and use that understanding to grow from a frontend background into fullstack work. Every other folder in this workspace was a small, clean teaching project built to demonstrate one idea. This one is the opposite on purpose, real, messy, inconsistent in places, and much larger, because that is what an actual job looks like.

## How to read this folder

If you want a real, paced study plan instead of just a table of contents, read [09-how-to-study-this-as-a-beginner.md](09-how-to-study-this-as-a-beginner.md) first, it turns everything below into a session by session plan with a prerequisite check and concrete things to actually do, not just read.

Otherwise, read the six root level files first, in order, they are the architectural spine every feature module in this app is built on top of.

1. [01-what-is-this-product.md](01-what-is-this-product.md), what this business actually is and how to tell, from the evidence in the code, without being told.
2. [02-high-level-architecture-and-bootstrap.md](02-high-level-architecture-and-bootstrap.md), `main.ts` and `app.module.ts`, read closely, and what almost ninety imported feature modules actually looks like in practice.
3. [03-configuration-and-secrets.md](03-configuration-and-secrets.md), why this app does not use a plain `.env` file, and how AWS Secrets Manager actually supplies its configuration.
4. [04-database-typeorm-and-repositories.md](04-database-typeorm-and-repositories.md), TypeORM, real migrations instead of automatic schema sync, and the shared base repository pattern.
5. [05-logging-and-observability.md](05-logging-and-observability.md), structured, shippable logging to AWS CloudWatch, and why that is a different skill than `console.log`.
6. [06-infrastructure-and-deployment.md](06-infrastructure-and-deployment.md), Terraform, AWS ECS, and everything that has to be true about the outside world before this app can serve one real request.

Then go into whichever business area interests you most, inside the `clusters` folder, each one is fully self contained with its own index.

7. [clusters/01-auth-and-identity](clusters/01-auth-and-identity/00-README.md), password login, Google and Twitter OAuth, Web3 wallet login, email verification, and role based access control.
8. [clusters/02-user-and-profile](clusters/02-user-and-profile/00-README.md), the core user account, linked wallet addresses, public builder profiles, and the waitlist.
9. [clusters/03-commerce-and-marketplace](clusters/03-commerce-and-marketplace/00-README.md), carts, checkout, coupons, invoices, refunds, and the secondary marketplace for reselling already owned domains.
10. [clusters/04-domain-core-product](clusters/04-domain-core-product/00-README.md), the single largest module in the system, what a domain actually is here, search, purchase, and the handoff to a real blockchain.
11. [clusters/05-blockchain-infrastructure](clusters/05-blockchain-infrastructure/00-README.md), a real Web3 fundamentals primer plus how this app talks to smart contracts, Alchemy, and Moralis.
12. [clusters/06-chain-and-registrar-integrations](clusters/06-chain-and-registrar-integrations/00-README.md), seventeen integrations with external chains and registrars, read as one shared pattern rather than seventeen separate stories.
13. [clusters/07-admin-cms-and-marketing](clusters/07-admin-cms-and-marketing/00-README.md), the admin panel, blog, landing pages, and real Google Analytics and Search Console integrations.
14. [clusters/08-affiliate-and-loyalty](clusters/08-affiliate-and-loyalty/00-README.md), the affiliate program and a genuinely real Web3 daily check in, reputation, and perks system.
15. [clusters/09-events-and-webhooks](clusters/09-events-and-webhooks/00-README.md), this app's internal event system versus the external webhooks it receives from payment providers, and how well each one actually verifies its sender.
16. [clusters/10-ai-and-infra-utilities](clusters/10-ai-and-infra-utilities/00-README.md), a real, safely guarded Claude powered domain name suggestion feature, plus the shared utilities every other cluster quietly depends on.
17. [clusters/11-payment-gateway-integrations](clusters/11-payment-gateway-integrations/00-README.md), Stripe, Coingate, and Cryptomus, and a direct comparison of how card payments and crypto payments actually get confirmed differently.
18. [clusters/12-marketplace-v2](clusters/12-marketplace-v2/00-README.md), new in the October 2026 update, the rebuilt secondary marketplace: wallet proof carried inside the JWT, Seaport style signed orders on Polygon paid in USDT, an on chain event poller, cursor pagination, an OpenSea market data pipeline, the ethers v6 migration, and the much larger test suite. Read clusters 03 and 05 before this one.

Finish with the two closing files, written after every cluster above was complete, tying the whole thing together.

19. [07-security-and-quality-findings-index.md](07-security-and-quality-findings-index.md), every real bug, gap, and inconsistency found across all twelve clusters, collected into one ranked list.
20. [08-frontend-to-fullstack-roadmap.md](08-frontend-to-fullstack-roadmap.md), what to actually do next with everything you just read.

## A note on how this folder was built

Eleven separate business domains this large could not be read carefully by one person, or one AI session, in a single continuous pass without either running out of time or losing depth. So this analysis was done by splitting the roughly ninety feature modules into eleven themed clusters and having each one read completely and independently, in parallel, the same way a real engineering team divides up a large onboarding or audit task among several people rather than one person trying to read everything alone. The six root level files and the two closing files were written directly, by hand, tying everything together once every cluster's findings were in. If you notice two cluster notes describing a shared piece of code slightly differently, that is worth treating as a real signal worth double checking against the actual source, not as a mistake to silently ignore, exactly the same instinct this whole folder is trying to teach.

## Update history, and how this folder stays in sync

The original notes were written on 10 September 2026 against commit `a131b429` on `uat` (the merge of pull request #943, `feat/t49-b02-jwt-wallet`). They were brought up to date on 2 October 2026 against `uat` merge commit `dc1ba3e8` (pull request #968, `feature/open-sea-api-integration`). That covers forty five commits, 219 changed files, and roughly 19,600 added lines. As before, the codebase itself was only read during this update, never modified, checked out, or run.

Here is what that update changed in this folder, so you know which notes to reread if you had already studied the earlier version.

The biggest change by far is the new [clusters/12-marketplace-v2](clusters/12-marketplace-v2/00-README.md), thirteen files covering the brand new `src/components/marketplacev2` module, about seventeen thousand of those new lines on its own. It adds six new database tables, more than thirty five new endpoints, two new `setInterval` style background loops (the event poller and the listing expiry sweep), four new `@Cron` market data jobs, and more than twenty new configuration keys in the AWS secret.

Second, `ethers` moved from v5 to v6 across the whole backend (commits `71fd2fec` and `26b1f0e4`), which changed the code in every blockchain related note. The affected notes in clusters 01, 03, 05 and 06 have corrected code excerpts, and [clusters/12-marketplace-v2/12-ethers-v6-migration-and-the-new-test-suite.md](clusters/12-marketplace-v2/12-ethers-v6-migration-and-the-new-test-suite.md) has a translation table.

Third, `CronModule` is now commented out in `app.module.ts` (commit `836d5f89`), so the legacy scheduled jobs it owned no longer run on `uat`. The [cron note in cluster 10](clusters/10-ai-and-infra-utilities/05-cron-scheduled-jobs.md) lists exactly which ones.

Fourth, the access token now carries optional `walletAddress` and `walletVerifiedAt` claims, refresh forwards them, and wallet login rotates its nonce. The auth notes in [cluster 01](clusters/01-auth-and-identity/00-README.md) were corrected for this.

Fifth, the test suite roughly doubled. The repository had 86 `*.spec.ts` files and about 594 test cases at `a131b429`, and it now has 148 spec files and about 1,280 test cases. Fifty of the new spec files are inside marketplace v2, and the rest are spread across auth, web3 auth, deployment verification, contract deployment, NFT collections, renewals, GM check in, and recent domains. None of them run automatically yet: neither CodeBuild nor the husky hooks call Jest.

Sixth, the actual stack on `uat` is NestJS 10, TypeORM 0.3 and ethers 6. The project `CLAUDE.md` still lists older versions, and still says there are no spec files at all, so trust the code (and these notes) over that file.

Every existing note that was edited ends with a section titled "Update from the October 2026 uat pull" describing exactly what changed, so you can jump straight to it. Root files 01 through 09 were all updated in the same way, and 02 also corrects an older mistake about rate limiting.

To update this folder again after a future pull, start from `dc1ba3e8`. Run `git log dc1ba3e8..uat` and `git diff --stat dc1ba3e8 uat` in the real codebase to see what changed, then go through the notes that mention each changed file.
