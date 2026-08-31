# NestJS Course Notes — Study Guide

This folder holds a full set of topic by topic notes built from two sources in this repo: the slide decks inside the `slides` folder, and the actual working code inside `src`. Every concept explained here points back to a real file in this project so you can open the code side by side with the note and see the idea in action instead of memorizing it in the abstract.

You are a beginner in NestJS and backend development, so every file is written assuming no prior backend knowledge. Each note explains what the concept is, why it exists, how the syntax works piece by piece, and how it connects to the rest of the request lifecycle. Code blocks are pulled directly from this project unless stated otherwise.

## How to read these notes

Go in order the first time through. Each file builds on the one before it. After the first pass, use this folder as a reference and jump straight to the topic you need.

1. [01-introduction-to-nestjs.md](01-introduction-to-nestjs.md) — what NestJS is, why it exists, what problem it solves compared to plain Express.
2. [02-project-setup-and-structure.md](02-project-setup-and-structure.md) — how this project is put together, `package.json`, `main.ts`, folder layout, npm scripts.
3. [03-architecture-overview.md](03-architecture-overview.md) — the big picture: client, controller, service, provider, module, decorators, and how a request flows through all of them.
4. [04-modules.md](04-modules.md) — the `@Module()` decorator, feature modules versus the root `AppModule`, imports/controllers/providers.
5. [05-controllers.md](05-controllers.md) — routing, `@Controller()`, `@Get/@Post/@Put/@Patch/@Delete`, reading route params and the request body.
6. [06-services-and-providers.md](06-services-and-providers.md) — `@Injectable()`, business logic, what a "provider" really means.
7. [07-dependency-injection.md](07-dependency-injection.md) — how NestJS wires classes together automatically, constructor injection, the IoC container.
8. [08-dto-and-interfaces.md](08-dto-and-interfaces.md) — shaping and validating data with DTOs and TypeScript interfaces.
9. [09-rest-api-and-http-methods.md](09-rest-api-and-http-methods.md) — what REST actually means and how this project implements a full CRUD API.
10. [10-validation-and-pipes.md](10-validation-and-pipes.md) — built-in pipes, `ValidationPipe`, and writing your own custom pipe.
11. [11-guards-and-authorization.md](11-guards-and-authorization.md) — protecting routes, `CanActivate`, custom decorators, and role-based access with `Reflector`.
12. [12-middleware.md](12-middleware.md) — logging and request preprocessing before a request ever reaches a controller.
13. [13-exception-filters.md](13-exception-filters.md) — catching errors in one centralized, consistent place.
14. [14-lifecycle-events.md](14-lifecycle-events.md) — hooks that run when a module starts up or the app shuts down.
15. [15-environment-variables-and-config.md](15-environment-variables-and-config.md) — `@nestjs/config`, `.env` files, and `ConfigService`.
16. [16-mongodb-and-nosql-intro.md](16-mongodb-and-nosql-intro.md) — conceptual introduction to MongoDB and NoSQL (this repo does not yet wire up a real database, so this note tells you exactly what is missing and what the next step would look like).
17. [17-testing-in-nestjs.md](17-testing-in-nestjs.md) — how the `.spec.ts` files and the e2e test in this repo work.
18. [18-request-lifecycle-recap.md](18-request-lifecycle-recap.md) — the whole journey of a single HTTP request through every layer, plus a map of every folder in `src` to the concept it teaches.

## A note on the source material

The slides in this repo are short, high level bullet decks (TechZeen's NestJS course). The actual depth in these notes comes from combining those bullet points with the working code in `src`, which is far more detailed than the slides alone. Where the code in this repo does something slightly different from the "textbook" way of doing it, that is called out explicitly so you understand both the theory and this project's specific choices.
