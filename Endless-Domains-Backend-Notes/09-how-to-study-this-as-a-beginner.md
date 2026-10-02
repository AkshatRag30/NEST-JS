# 09. How to Study This Folder as a Beginner

## Who this is for

This is for you specifically, a frontend developer at Endless Domains who wants to actually understand the backend you already work alongside, not a hypothetical junior engineer. That matters for how you should use this folder. This is not a course with no stakes, it is notes about the real system your product runs on, so treat anything confusing as a real question worth asking a real backend teammate, and treat anything flagged as a security concern in [07-security-and-quality-findings-index.md](07-security-and-quality-findings-index.md) as something worth actually raising, not just filing away as a learning example.

## Before you open a single cluster, an honest self check

A little over a hundred files (one hundred and nine since the October 2026 update) is a real amount of material, and this folder assumes you already have some NestJS fundamentals under you. Before starting, ask yourself three questions honestly. Do you know what a module, a controller, a service, and dependency injection are, and could you explain what happens when a request hits a `@Get()` route without looking anything up. Do you know, at least roughly, what a foreign key, a one to many relationship, and a many to many relationship mean in a relational database. Have you ever heard of a blockchain, a smart contract, or a wallet address, even if you have never touched one yourself.

If the answer to the first question is genuinely no, stop here and go read `Complete-Nest-JS-Full-Course-2025-The-Techzeen-main/notes` first, at least files 01 through 09 in that folder, this whole codebase assumes that foundation and nothing here will re teach it to you. If the answer to the second question is shaky, skim `PostgreSQL-with-NEST-JS-main/notes`, specifically the entity and relationship notes, before you reach the database heavy clusters here. If the answer to the third question is no, that is completely fine and expected, [clusters/05-blockchain-infrastructure/01-blockchain-fundamentals-primer.md](clusters/05-blockchain-infrastructure/01-blockchain-fundamentals-primer.md) was written specifically assuming zero prior Web3 knowledge, just make sure you actually read that one and do not skip ahead past it.

## The one rule that matters more than any schedule

Never read a note in isolation from the real code it is describing. Every single note in this folder names exact file paths inside `D:\Akshat Endless\backend\backend`. Open that real file next to the note, in a second editor tab or window, every time. The notes are a guide for reading the code, not a replacement for reading the code, and the actual skill you are building, reading a real codebase confidently, only forms if you keep doing that side by side check every single time, even when it feels slower at first. It gets faster quickly.

## A realistic pace, spread over roughly three to four weeks of part time study

Do not try to do this in one sitting, and do not feel behind if it takes longer than this. This is a rough shape, not a deadline.

Session 1. Read [00-README.md](00-README.md) in full so you know what exists, then read [01-what-is-this-product.md](01-what-is-this-product.md) and [02-high-level-architecture-and-bootstrap.md](02-high-level-architecture-and-bootstrap.md), with `src/main.ts` and `src/app.module.ts` open next to them the whole time. Stop there for the day. This session alone is enough to change how you read the rest of the codebase at work tomorrow.

Session 2. Read [03-configuration-and-secrets.md](03-configuration-and-secrets.md), [04-database-typeorm-and-repositories.md](04-database-typeorm-and-repositories.md), and [05-logging-and-observability.md](05-logging-and-observability.md), each with its real source files open. By the end of this session you should be able to answer, from memory, where a database password actually comes from in this app, and why `synchronize` is set to false.

Session 3. Read [06-infrastructure-and-deployment.md](06-infrastructure-and-deployment.md) with `terraform/main.tf` open next to it. You are not expected to become a Terraform expert from one file, just walk away understanding the shape, a load balancer, a container service, a database, and a domain name all have to exist and agree with each other before the app you write TypeScript for actually serves a real request.

Sessions 4 through 14, roughly one cluster per session, in this order, since later clusters lean on concepts earlier ones already explained. [clusters/01-auth-and-identity](clusters/01-auth-and-identity/00-README.md), then [02-user-and-profile](clusters/02-user-and-profile/00-README.md), then [04-domain-core-product](clusters/04-domain-core-product/00-README.md) (the single most important one, give it two sessions if you need to, not one), then [05-blockchain-infrastructure](clusters/05-blockchain-infrastructure/00-README.md), then [06-chain-and-registrar-integrations](clusters/06-chain-and-registrar-integrations/00-README.md), then [03-commerce-and-marketplace](clusters/03-commerce-and-marketplace/00-README.md) (also worth two sessions), then [11-payment-gateway-integrations](clusters/11-payment-gateway-integrations/00-README.md), then [09-events-and-webhooks](clusters/09-events-and-webhooks/00-README.md), then [08-affiliate-and-loyalty](clusters/08-affiliate-and-loyalty/00-README.md), then [07-admin-cms-and-marketing](clusters/07-admin-cms-and-marketing/00-README.md), then [10-ai-and-infra-utilities](clusters/10-ai-and-infra-utilities/00-README.md), then, as its own block of sessions described in the update section at the end of this file, [12-marketplace-v2](clusters/12-marketplace-v2/00-README.md).

Final session. Read [07-security-and-quality-findings-index.md](07-security-and-quality-findings-index.md) end to end in one sitting, now that you have the context to actually understand every entry, and then [08-frontend-to-fullstack-roadmap.md](08-frontend-to-fullstack-roadmap.md) to turn all of it into your own next steps.

## What to actually do in each session, not just reading

Before you open a cluster's own notes, open the real folder it covers in the actual codebase and spend five minutes alone with just the file names and the entity files, and guess, in your own words, what you think this module does. Then read the cluster's `00-README.md` and see how close you were. Being wrong is the useful part, not a failure, it tells you exactly which of your assumptions about how backends work needs correcting.

While reading each numbered file, whenever it quotes a real function or method, stop and try to predict what it returns or what it does to the database before reading the explanation underneath it. When you reach a cluster note that describes a real finding, a bug, a piece of dead code, an inconsistency between two similar features, go verify it yourself in the real file before taking the note's word for it. That verification habit is the entire point of this exercise, and it is exactly what a senior engineer does automatically, confirm a claim against the source rather than trusting a summary, even a careful one.

At the end of each cluster, close the notes and write, from memory, in three or four sentences of your own, what that business area actually does and how it connects to at least one other cluster you have already read. If you cannot do that, reread the cluster's `00-README.md` before moving on, do not push forward on a shaky foundation, everything after the domain and commerce clusters leans on them.

## When something genuinely does not make sense

Grep the real codebase yourself for the class or function name the note mentions, and read every place it gets used, not just the one file the note pointed you at. If it still does not make sense after that, that is a real, legitimate question, take it to an actual backend engineer on your team, phrased as specifically as you can, not "how does auth work" but "I don't understand why the Solana wallet login in web3-auth.service.ts does not check a signature the way the Ethereum one does, is that intentional." Asking a question shaped that specifically, backed by real code you actually read, is exactly the kind of question that makes a team take a frontend developer moving into fullstack work seriously.

## The actual finish line

You are done with this folder, for now, not forever, when you can pick any cluster you already read, close every note, and produce your own short version of it from the real code alone. That is the same closing test [08-frontend-to-fullstack-roadmap.md](08-frontend-to-fullstack-roadmap.md) ends on, and it is the real skill this entire folder exists to build in you, not memorized facts about Endless Domains specifically, but confidence opening any unfamiliar, real codebase at all.

## Update from the October 2026 uat pull

The update that brought this folder in line with `uat` commit `dc1ba3e8` added a whole new cluster, [12-marketplace-v2](clusters/12-marketplace-v2/00-README.md), and edited about twenty existing notes. Here is how to fit that into the plan above, depending on where you are.

### If you have not started yet

Follow the plan above as written. When you reach a note that ends with "Update from the October 2026 uat pull", read that section straight away as part of the same session, not later, since it corrects or extends what you just read. Then add the four sessions below after cluster 10 and before the final findings session.

### If you already finished the earlier version

Start with a short catch up session. Read the "Update history" section of [00-README.md](00-README.md), then the update sections at the bottom of root files 02, 03, 04 and 05, with `src/main.ts`, `src/app.module.ts`, `run/deploy.sh` and `package.json` open beside them. By the end you should be able to explain from memory why `ThrottlerModule` does not throttle most routes, why six new tables appeared without a migration file, and which scheduled jobs stopped running. Then reread the update sections in clusters 01, 03, 05, 06, 08 and 10, which are mostly corrected ethers v6 code excerpts, and do the four sessions below.

### Four sessions for cluster 12

Cluster 12 assumes you already understand the v1 marketplace in cluster 03 and the Web3 primer in cluster 05. If either feels shaky, reread [clusters/03-commerce-and-marketplace/03-domain-listing-and-buy-domain-secondary-marketplace.md](clusters/03-commerce-and-marketplace/03-domain-listing-and-buy-domain-secondary-marketplace.md) and [clusters/05-blockchain-infrastructure/01-blockchain-fundamentals-primer.md](clusters/05-blockchain-infrastructure/01-blockchain-fundamentals-primer.md) first.

Session A, the map and identity. Read cluster 12 notes 00, 01 and 02 with `src/components/marketplacev2/marketplacev2.module.ts`, `config/chain-config.loader.ts`, `wallet-verification/` and `src/@core/common/guards/require-verified-wallet.guard.ts` open. Then do the exercise. Write down, in order, every HTTP call a React app makes from "user logs in with email" to "user is allowed to create a listing", including which headers go with each call and which error code means "go sign the wallet challenge again".

Session B, orders. Read notes 03, 04 and 05 with `order/entity/order.entity.ts` and `order/order.service.ts` open. Seaport is the hardest new idea in this whole folder, so take it slowly. Before reading note 04, try to list for yourself what the backend must check before it agrees to store a signed listing, then compare your list with the twelve checks in the note. Finish by explaining to yourself, in plain words, why "cancelled" in this database does not mean "cannot be bought", because that single idea is the most important thing in the cluster.

Session C, reading data. Read notes 06 and 07 with `order/util/order-cursor.util.ts`, `order/util/domain-category.util.ts` and `listing-status/` open. Then do the exercise. Sketch the React infinite scroll for the browse page, using `nextCursor`, and decide what your component should do when the user changes the sort order (the right answer is to throw away the cursor and start over, and note 06 explains why the backend returns a 500 if you do not).

Session D, background work. Read notes 08 and 09, then 10 and 11, with `poller/tick/chain-event-source-poller.service.ts` and `market-data/market-data.scheduler.ts` open. These are the first real background workers in this codebase, so they are worth studying as patterns. As you read, keep one question in mind for every piece of state: what happens if two copies of this process run at once, and what happens after a restart. Then read note 12 on the ethers v6 migration and the test suite, and run `git ls-files '*.spec.ts' | wc -l` in the real codebase yourself to see the number.

### A new kind of exercise this cluster makes possible

The older clusters had almost no tests to learn from. Marketplace v2 has fifty spec files. For any service you read in sessions B to D, open its `.spec.ts` beside it and read the test titles before the implementation. Test titles such as `TC2.3: passes through and attaches req.walletAddress for a fresh, valid claim` are often the clearest written statement of what the code is meant to do. Then pick one finding from the update section of [07-security-and-quality-findings-index.md](07-security-and-quality-findings-index.md) and work out which test would have caught it, which is a very good way to start thinking like the person who reviews pull requests rather than only the person who writes them.
