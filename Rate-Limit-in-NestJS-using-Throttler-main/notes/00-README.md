# Rate Limiting in NestJS — Study Guide

This folder holds a full set of notes built from the actual code in this repository. Unlike a large multi feature course, this project is small and focused on exactly one idea: how to add rate limiting to a NestJS application using the official `@nestjs/throttler` package. The whole application is a single module, a single controller with one route, and a single service, and every line of it exists to demonstrate throttling, nothing more.

Because the project is so small, these notes are shorter than the notes you might find in a bigger course repo, but they go just as deep on the one thing this project actually teaches. Every code block below is copied directly from the real files in `src` and `test`, and every explanation is checked against what the code actually does, not what a tutorial usually says it should do.

## How to read these notes

Go in order the first time through, since each note builds on the one before it.

1. [01-project-setup-and-bootstrap.md](01-project-setup-and-bootstrap.md) — what this project is, `package.json`, `main.ts`, the TypeScript and Nest CLI configuration, and the tooling around it.
2. [02-what-is-rate-limiting-and-throttlermodule.md](02-what-is-rate-limiting-and-throttlermodule.md) — what rate limiting means and why an API needs it, then `ThrottlerModule.forRoot()` in `app.module.ts` piece by piece.
3. [03-global-guards-and-app-guard.md](03-global-guards-and-app-guard.md) — how `ThrottlerGuard` gets applied to every single route in the app through the `APP_GUARD` token, without a single `@UseGuards()` anywhere in the code.
4. [04-per-route-throttle-overrides.md](04-per-route-throttle-overrides.md) — the `@Throttle()` decorator on the one route in `app.controller.ts`, how a route level override interacts with the global config, and a real inconsistency in how time is written in this codebase.
5. [05-testing-and-recap.md](05-testing-and-recap.md) — a close, honest read of both spec files in this repo, a real bug that makes both of them fail as written, and a full recap of how one request travels through this app from arrival to response.

## A note on the source material

There are no slides or lecture PDFs in this repository, only the working code, so these notes are built entirely by reading `src/app.module.ts`, `src/app.controller.ts`, `src/app.service.ts`, `src/main.ts`, `src/app.controller.spec.ts`, and `test/app.e2e-spec.ts` directly, alongside the supporting configuration files. Where the code has a genuine inconsistency or a test that would actually fail if you ran it, that is called out plainly rather than smoothed over, the same way you would want a senior teammate to point it out to you during a real code review.
