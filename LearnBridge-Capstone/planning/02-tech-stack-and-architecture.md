# Tech Stack and Architecture

## The core framework

NestJS, the same framework every one of the eight source folders used, structured the same way you already learned it in `Complete-Nest-JS-Full-Course-2025-The-Techzeen-main`, modules containing controllers and providers, wired together by dependency injection. Nothing about the framework itself is new here, what is new is using it at a scale where module boundaries actually matter, with roughly ten feature modules instead of one or two.

## Two databases, on purpose, not by accident

PostgreSQL, through Prisma, is the primary store, holding users, courses, categories, enrollments, orders, and reviews. This is the relational, structured half of the product, where every relationship needs to be enforced by the database itself through foreign keys, and where a payment and an enrollment must succeed or fail together. Prisma was chosen over TypeORM (the ORM used in the PostgreSQL source folder) as the primary tool because Prisma's schema file and generated client are the more modern, currently favored approach in the Node ecosystem, and because you already have a working reference for it in `PrismaORM-NeonDB-NestJS-main`. TypeORM is not skipped entirely though, see [14-phase-10-PRD-typeorm-comparison-appendix.md](14-phase-10-PRD-typeorm-comparison-appendix.md), an optional appendix phase where you rebuild one small piece of the system with TypeORM specifically so you can feel the repository pattern and the decorator based entity style in your own hands, not just read about it.

MongoDB, through Mongoose, holds exactly one thing, notifications, and is chosen for that specific piece on purpose rather than out of habit. A notification is high volume, short lived, has a loosely defined shape depending on its type, and is never joined against in a complex way, you just fetch the ones belonging to one user. That is precisely the profile of data MongoDB is good at, and precisely the profile the relational half of the system is not built for. Running two databases in one project is a deliberate, realistic constraint, plenty of real companies run a relational store next to a document store for exactly this reason, one for the data that must be perfectly consistent and structured, one for the data that is high volume and loosely shaped.

## Two API styles

REST is the primary interface, covering every feature in the product. GraphQL, through `@nestjs/graphql` with the Apollo driver, mirrors the read side of the course catalog and reviews specifically, the same combination already proven out in `GraphQL-with-NestJS-main` and `PrismaORM-NeonDB-NestJS-main`. The point of building both is not that a real product usually needs both, most don't, the point is that building the same underlying data through two different API shapes is the fastest way to actually understand what a resolver is doing differently from a controller method, instead of only reading about the difference.

## Authentication

JSON Web Tokens, signed and verified with `@nestjs/jwt` and validated on incoming requests with `passport-jwt` through `@nestjs/passport`, exactly the stack already proven out in `JWT-Auth-with-Mongo-DB-Nest-JS-main`, adapted here to sit on top of a Prisma backed `User` table instead of a Mongoose schema. Passwords are hashed with `bcrypt` before ever touching the database. The JWT secret is read through `ConfigService`, never hardcoded, and its absence is treated as a startup failure, not a runtime surprise, see [05-phase-1-PRD-foundations-and-config.md](05-phase-1-PRD-foundations-and-config.md) for exactly how.

## Cross cutting concerns

A single global `ValidationPipe`, registered once in `main.ts`, replaces the pattern of accepting loosely typed `Partial<Something>` bodies seen in a few of the reference projects, every DTO in LearnBridge is a real class with real `class-validator` decorators, and the pipe is what actually makes those decorators do something, closing the exact gap found sitting inert in `GraphQL-with-NestJS-main`'s DTOs. A single global exception filter normalizes every error response into one consistent shape. A single logging middleware records method, path, and response time for every request. Rate limiting, through `@nestjs/throttler`, is applied globally with one sensible default and then tightened specifically on the authentication routes, following the pattern from `Rate-Limit-in-NestJS-using-Throttler-main`, but this time with an override that actually does something different from the global default, unlike that reference project's identical override.

## Testing

Jest, the same test runner every reference project already used, but used correctly this time. Every unit test that needs a Prisma client, a Mongoose model, or another service, gets a real, explicit mock or stub supplied to the testing module, rather than an empty `providers` array and a constructor NestJS cannot actually satisfy, which is the single most common failure pattern found across all eight reference projects' spec files. End to end tests run against a real, disposable test database rather than the primary one.

## Folder shape, once code actually starts

Once you begin implementing, `src` will hold one folder per feature module, `auth`, `users`, `courses`, `categories`, `enrollments`, `orders`, `reviews`, `notifications`, `graphql` (or the GraphQL pieces living alongside `courses`/`reviews` directly, your call when you get there), plus `prisma` and `common` folders for the Prisma service and shared pipes, filters, guards, and middleware respectively. This mirrors the shape you already saw across the reference projects, just with more feature modules than any single one of them had on its own.
