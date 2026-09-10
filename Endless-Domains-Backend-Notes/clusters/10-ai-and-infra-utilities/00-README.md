# 10. AI Features and Shared Infrastructure Utilities

## What this cluster covers

This slice of the app splits cleanly into two halves that do not talk to each other very much, which is exactly why they are documented together rather than each getting its own cluster.

The first half is genuinely new product surface built on top of everything the other clusters already documented, the actual AI features. Endless Domains talks to Anthropic's Claude in three quite different ways, a free, unauthenticated domain name idea generator aimed at visitors who have not signed up yet, a paid, per-user Domain Advisor that answers questions about domains a signed in user already owns, and an internal admin assistant that lets a superadmin ask plain English questions about orders, users, and platform health instead of writing SQL by hand. All three are real, live, and wired into `app.module.ts`.

The second half is infrastructure plumbing that almost every other feature module leans on without necessarily being aware of it, a health check endpoint for load balancers, an admin panel over which blockchain domain providers and TLDs are configured, two different AWS wrappers (S3 file storage and SNS notifications) that sit alongside the `SecretsService` already covered in note 03, a Web3 decentralized storage integration (IPFS, explained here from first principles), and the cron jobs that quietly run in the background keeping domains, emails, and mint statuses moving.

Underneath both halves are the shared `@core` utilities every other cluster's code depends on without writing its own version, a retry helper for flaky third party calls, a shared HTTP client, the mail sending pipeline that sits behind every "you've got mail" feature in the whole app, the event system that decouples a feature module from having to know how an email actually gets sent, and the base entity class that almost every database table in this application quietly extends.

## Files in this cluster

`01-ai-domain-discovery.md` covers the unauthenticated, visitor facing AI feature that turns a business idea into candidate domain names, including its rate limiting and cost circuit breaker.

`02-ai-domain-advisor-and-subscription.md` covers the signed in, paid Domain Advisor, its free query quota, its Stripe based subscription flow, and the webhook that actually activates a subscription.

`03-ai-admin-tools-and-insights.md` covers the internal admin side of the AI module, the tool registry and dispatcher architecture that both the advisor and the admin assistant share, the dashboard insights summary, the downloadable AI business report, and the scheduled alert checker.

`04-infra-health-provider-aws-ipfs.md` covers the health check endpoint, the provider management admin panel, the S3 and SNS AWS wrappers, and the IPFS integration, explained from the ground up for a reader who has never touched decentralized storage before.

`05-cron-scheduled-jobs.md` covers every `@Cron` decorated method actually running in this codebase, what each one does, and how often.

`06-shared-utilities-retry-http-mail-events.md` covers the retry helper, the shared HTTP client, the mail service and its event driven trigger system, and the email address normalizer.

`07-shared-core-building-blocks.md` covers the base entity class nearly every table extends, the shared response and pagination DTOs, the shared enums, and the custom validation decorators and pipes used across the whole app.

## The one thing worth carrying forward

Almost every file in the AI half of this cluster constructs its own `new Anthropic(...)` client inside its own constructor rather than sharing one instance, and almost every file in the infrastructure half of this cluster constructs its own `new S3Client(...)` or repeats the exact "fetch a secret bundle and cache it" pattern that note 03 already flagged for `SecretsService`. None of this is broken, Anthropic's SDK and the AWS SDK are both cheap to instantiate and thread safe to use this way, but noticing the repetition is a genuinely useful skill, and it is the same skill note 03 and note 04 were already trying to teach with `SecretsService` and the base repository.
