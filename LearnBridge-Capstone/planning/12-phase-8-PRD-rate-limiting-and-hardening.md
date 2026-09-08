# Phase 8 PRD: Rate Limiting and Hardening

## Goal

Apply `@nestjs/throttler` globally the way `Rate-Limit-in-NestJS-using-Throttler-main` already showed you, but this time make the per route override on the authentication endpoints actually different from the global default, which is the one thing that reference project's own override failed to do.

## Concepts practiced

`ThrottlerModule.forRoot`, registering `ThrottlerGuard` globally through the `APP_GUARD` token, and `@Throttle()` overrides that are deliberately stricter than the default rather than accidentally identical to it.

## Scope

Register `ThrottlerModule.forRoot` in `AppModule` with a sensible general purpose default, for example twenty requests per minute per IP, covering every route in the product by default through the same `APP_GUARD` pattern already used for `JwtAuthGuard`'s sibling registration in phase 1, if you choose to also register that guard globally there, or applied per controller if you decided against a global auth guard, your call to make consistent with how phase 2 was actually built. Apply a stricter `@Throttle()` override specifically to `POST /auth/login` and `POST /auth/signup`, for example five requests per minute per IP, meaningfully tighter than the general default, specifically to blunt credential stuffing and brute force attempts against those two routes.

While in this phase, also review and tighten a few other cross cutting concerns that belong here rather than in any single feature module. Confirm CORS is configured deliberately in `main.ts` rather than left at a wide open default. Confirm the global exception filter from phase 1 never leaks a raw stack trace or an internal error message to the client in production, only in a development log. Confirm every route that should require authentication actually has a guard on it, by writing a short script or manually walking every controller and resolver in the project and checking it against the API surface list in every earlier phase's PRD, this is exactly the check that would have caught the Supabase guard sitting completely unapplied in the PostgreSQL reference project.

## API surface

No new routes, this phase is entirely about what wraps around the routes that already exist.

## Acceptance criteria

1. Sending six rapid login attempts from the same client within a minute results in the sixth one receiving a 429, with a clear rate limit error message, while a request to an unrelated, unthrottled endpoint made immediately after still succeeds.
2. Sending twenty five rapid requests to any ordinary authenticated endpoint within a minute results in requests past the global limit receiving a 429 as well, confirming the global default genuinely applies everywhere, not only to the two routes with an explicit override.
3. A full audit against the API surface list from every earlier phase confirms every route that should be protected by `JwtAuthGuard`, a role check, or both, actually has them, with no gaps.
4. A deliberately thrown unexpected error in a development environment shows a full error in the server log but only a generic message in the HTTP response body.

## Explicit trap to avoid

Do not set the login and signup override to the exact same limit as the global default, the mistake made in the reference project, where the per route override was functionally invisible. The whole point of an override is that it does something the default does not, verify the numbers are actually different before considering this phase done.
