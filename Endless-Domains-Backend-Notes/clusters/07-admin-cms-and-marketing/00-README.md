# Cluster 07: Admin Tooling, Content Management, and Marketing and SEO Integrations

## What this cluster actually covers

Eleven folders sit under this cluster's umbrella, and reading every file in all of them rather than trusting the folder names alone turned up a handful of real corrections worth flagging before anything else. `blog` is not a content management system with posts, categories, and authors, it is purely a view counting and analytics service for blog posts that must live somewhere else entirely, most likely a separate headless CMS or a static frontend, since no post entity exists anywhere in this folder. `localization` is not a table of pre translated strings, it is a live, on demand machine translation service backed by Azure Translator, called at read time and cached afterward. `analytics` (the plain, undecorated folder) is not connected to Google Analytics at all, it is a tiny, generic, unauthenticated event sink that frontend code posts arbitrary click and cart events into. `search-management` has nothing to do with Google's Search Console despite the name sitting one folder away from `search-console`, it tracks searches typed into Endless Domains' own domain search bar. And `ads` is not a Google or Meta advertising integration, it manages Endless Domains' own ad inventory shown on the actual decentralized websites its customers host at their Web3 domains. Every one of these corrections came from reading the real entities, controllers, and services, exactly the habit [01-what-is-this-product.md](../../01-what-is-this-product.md) argues a fullstack engineer needs to build on a codebase this size.

## The single most useful cross cutting finding

[02-high-level-architecture-and-bootstrap.md](../../02-high-level-architecture-and-bootstrap.md) already flagged a comment in the real `app.module.ts`, directly above the global `CacheModule.register(...)` line, saying the short lived in memory cache exists specifically for read heavy, slow changing dashboard, stats, summary, and insights endpoints that use a `CacheInterceptor`. This cluster is where that promise gets tested against real code, and the answer is genuinely mixed. `AdminAnalyticsController`'s `overview`, `ga4`, and `gsc` GET routes (file 06) use `@UseInterceptors(CacheInterceptor)` and an explicit five minute `@CacheTTL(300000)` exactly as advertised, and so do three of the newer endpoints on `SearchManagementController` (`search-trend`, `tld-chart`, `summary`, file 08). But the admin user management dashboard's own aggregate endpoints, registration trend and the global user counts block behind `getAllUsers`, use no caching at all despite being exactly the kind of slow, aggregate query the comment describes (file 01). The lesson worth carrying forward is that a codebase wide convention like this one is real, present, and used correctly in several places, but it is not applied uniformly, and checking each controller rather than assuming the pattern held everywhere is the only way to know which is true.

## The second cross cutting finding: authorization is inconsistent in a way worth noticing

Across this cluster alone, admin style write endpoints are protected by at least three different mechanisms that do not agree with each other. Some routes use `AccessTokenGuard` plus `SuperAdminAccessGuard`, which decodes and re verifies the JWT itself and checks for a `superAdmin` or `Admin` role. Others use `AccessTokenGuard` plus `AdminTokenGuard`, which does something completely different, a plain equality check against a static `admin-token` header value pulled from configuration, with no JWT involved at all. And several controllers, most notably the entire write side of `content-managment`'s static content endpoints, `faqs`, and `testimonials` inside `landing_page`, and the newer analytics endpoints on `search-management`, have no guard whatsoever. This is spelled out in detail in each relevant file below rather than only here, because the specific missing guard on a specific route is the part worth remembering.

## Reading order

[01-admin-user-management.md](01-admin-user-management.md) covers the one real capability living under `admin`, a full internal user management dashboard with search, filters, growth KPIs, CSV export, and moderation actions.

[02-static-content-and-tld-metadata.md](02-static-content-and-tld-metadata.md) covers `content-managment` (generic key and value static page content like terms and privacy) and `tlds-content-managment` (the structured content and metadata behind each top level domain's own landing page).

[03-localization-and-translation.md](03-localization-and-translation.md) covers the Azure Translator integration that the TLD content service calls into, and corrects the assumption that this folder holds static translated strings.

[04-blog-post-view-tracking.md](04-blog-post-view-tracking.md) covers `blog`, and corrects the assumption that it is a content management system.

[05-landing-page-marketing-content.md](05-landing-page-marketing-content.md) covers `landing_page`, the largest folder in this cluster at around sixty files across six nearly identical CRUD sub-features, and is where the authorization inconsistency above is most visible.

[06-google-analytics-integration.md](06-google-analytics-integration.md) covers `google-analytics` in depth, its service account based Google auth, its own outbound rate limiting against Google's real quotas, and its sync cron jobs.

[07-search-console-and-unified-admin-dashboard.md](07-search-console-and-unified-admin-dashboard.md) covers `search-console` and the small aggregating module that combines it with `google-analytics` into one admin dashboard controller.

[08-internal-search-analytics-and-event-tracking.md](08-internal-search-analytics-and-event-tracking.md) covers `search-management` (the site's own domain search analytics) and the plain `analytics` folder (the generic frontend event sink), two unrelated things that share a cluster only by proximity.

[09-ads-and-web3-advertising.md](09-ads-and-web3-advertising.md) covers `ads`, including a real, findable bug in one of its filter branches.
