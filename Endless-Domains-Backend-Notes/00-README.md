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

Finish with the two closing files, written after every cluster above was complete, tying the whole thing together.

18. [07-security-and-quality-findings-index.md](07-security-and-quality-findings-index.md), every real bug, gap, and inconsistency found across all eleven clusters, collected into one ranked list.
19. [08-frontend-to-fullstack-roadmap.md](08-frontend-to-fullstack-roadmap.md), what to actually do next with everything you just read.

## A note on how this folder was built

Eleven separate business domains this large could not be read carefully by one person, or one AI session, in a single continuous pass without either running out of time or losing depth. So this analysis was done by splitting the roughly ninety feature modules into eleven themed clusters and having each one read completely and independently, in parallel, the same way a real engineering team divides up a large onboarding or audit task among several people rather than one person trying to read everything alone. The six root level files and the two closing files were written directly, by hand, tying everything together once every cluster's findings were in. If you notice two cluster notes describing a shared piece of code slightly differently, that is worth treating as a real signal worth double checking against the actual source, not as a mistake to silently ignore, exactly the same instinct this whole folder is trying to teach.
