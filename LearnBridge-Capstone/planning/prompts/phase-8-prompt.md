# Phase 8 Prompt: Rate Limiting and Hardening

Use this once phase 7 is complete and both the REST and GraphQL surfaces are working against the same data.

Copy everything inside the fenced block below into Claude Code as one message.

```
Before writing any code, read planning/12-phase-8-PRD-rate-limiting-and-hardening.md completely.

First, inspect the current AppModule and every guard applied across the project so far, and tell me explicitly whether JwtAuthGuard was registered globally through APP_GUARD back in phase 2 or applied per controller instead, do not assume, check the actual code. Use whichever pattern is already in place as the model for how you register the new global throttler guard, so the project stays internally consistent rather than mixing two different styles for two similar concerns.

Install @nestjs/throttler. Register ThrottlerModule.forRoot in AppModule with one named throttler called default, using the seconds() helper for its ttl, not a raw millisecond number, set as seconds(60), with a limit of 20, and a clear custom errorMessage. Register ThrottlerGuard globally through the APP_GUARD provider token, the same technique already used elsewhere in this project for a different guard.

On the login and signup routes specifically in AuthController, add a @Throttle override using the same default throttler name, but with a limit of 5 over the same seconds(60) window, this number must be meaningfully and visibly stricter than the global default of 20, if you find yourself writing the same numbers as the global default, that is wrong, stop and fix it, that exact mistake, an override identical to the default, is a documented bug from an earlier reference project and must not be repeated here.

Configure CORS explicitly in main.ts using app.enableCors with a real origin configuration, reading an allowed origin list from an environment variable if one is set, and falling back to a sensible localhost default for development, rather than calling app.enableCors with no arguments at all.

Confirm the global exception filter from phase 1 never sends a raw stack trace or an internal error message in its response body, only a generic message, verified by deliberately throwing a plain, unhandled Error inside a temporary test route and inspecting the actual HTTP response body, then removing that temporary route.

Produce a written audit. Go through every controller and resolver method in the entire project, and produce a markdown table at docs/route-audit.md listing the HTTP method or GraphQL operation name, the path or field name, whether a JWT guard is applied, and whether a role restriction is applied, for every single one of them. Cross check this table against the API surface section of every phase's PRD from phase 2 through phase 7, and flag, explicitly and by name, any route that should be protected according to its PRD but currently is not.

When you are done, show me the route audit table in full, send six rapid login requests from the same client and show me the sixth one returning a 429, and send twenty five rapid requests to an ordinary authenticated route and show me one of the later ones also returning a 429 from the global default.
```
