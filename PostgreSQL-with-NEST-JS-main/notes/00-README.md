# PostgreSQL with NestJS — Study Guide

This folder holds a complete set of topic by topic notes built from this repository. Where the MongoDB sibling project in this same workspace teaches Mongoose and document modeling, this project teaches the relational side of the same NestJS skill set: TypeORM, a real PostgreSQL connection, the repository pattern, and a JWT based auth guard built around Supabase. The whole codebase is small, one slide deck called `PostgreSQL Introduction.pdf` in the `slides` folder, one real feature module called `employees`, one bare bones `user` module, and one authentication guard, but every piece of it is worth reading closely because it mixes solid working code with a few genuine bugs and gaps that are worth learning to spot.

Every note below explains a concept and then points at the exact file and lines in this repo where that concept is used, so you can open the code side by side with the note.

## How to read these notes

Go in order the first time. Each note builds on the one before it.

1. [01-project-setup-and-postgres-connection.md](01-project-setup-and-postgres-connection.md) — what this project is, `package.json`, `main.ts`, and how `app.module.ts` actually connects to PostgreSQL through `TypeOrmModule.forRoot`, including the missing `DATABASE_URL` environment variable.
2. [02-postgresql-and-relational-concepts-from-slides.md](02-postgresql-and-relational-concepts-from-slides.md) — everything in the `PostgreSQL Introduction.pdf` slide deck, what PostgreSQL is, why it is used, its SQL plus NoSQL hybrid features, and what TypeORM is, tied back to the real code.
3. [03-typeorm-entities-vs-mongoose-schemas.md](03-typeorm-entities-vs-mongoose-schemas.md) — `@Entity()`, `@Column()`, and `@PrimaryGeneratedColumn()` in `employees.entity.ts` and `user.entity.ts`, explained and directly compared to the `@Schema()`/`@Prop()` pattern from the MongoDB sibling project.
4. [04-employees-module-repository-and-service.md](04-employees-module-repository-and-service.md) — `employees.module.ts`, the repository pattern, `@InjectRepository()`, and the full body of `employees.service.ts`, including two real bugs in its error messages.
5. [05-employees-controller-routes.md](05-employees-controller-routes.md) — every route in `employees.controller.ts`, what each one does, and which single route is actually protected by a guard.
6. [06-user-entity-and-module-gap.md](06-user-entity-and-module-gap.md) — `user.entity.ts` and `user.module.ts`, and an honest look at why this module has no service, no controller, and no exports at all.
7. [07-supabase-auth-guard.md](07-supabase-auth-guard.md) — what Supabase is, how `SupabaseAuthGuard` verifies a JWT locally rather than calling out to Supabase's API, and a search confirming exactly where it is, and is not, actually applied.
8. [08-testing-gaps.md](08-testing-gaps.md) — every `.spec.ts` file in this repo checked for whether it would actually pass, including one spec file that would fail to even compile.
9. [09-full-recap-and-file-map.md](09-full-recap-and-file-map.md) — one real HTTP request traced from route to database and back, plus a table mapping every file in `src` to the note that explains it.

## A note on the source material

The slide deck in this repo is a single short deck, eight slides, mostly bullet points, introducing PostgreSQL and TypeORM at a conceptual level. All of the real depth in these notes comes from reading the actual working code in `src`, which is a small, honest teaching project rather than a finished production app. Where the code has a genuine bug, a missing piece, or a decision that only makes sense once you have read the MongoDB sibling project for comparison, that is called out plainly rather than glossed over.
