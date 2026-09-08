# Concept Coverage Map

This is the file that actually proves this plan covers everything, so it is worth reading slowly. Every row below is one concept you already have a note on, from one of the eight source folders, and exactly where in LearnBridge that same concept gets built for real. If a concept in your notes is missing from this table, that is a planning mistake, and this file should be corrected before you write a single line of code, not after.

## From Complete-Nest-JS-Full-Course-2025-The-Techzeen-main

| Note | Concept | Where it lives in LearnBridge |
|---|---|---|
| 01, 03 | What NestJS is, the client/controller/service/module architecture | Every module, from the very first one you write in phase 1 |
| 04 | `@Module()`, feature modules versus the root `AppModule` | Ten plus feature modules, listed in [02-tech-stack-and-architecture.md](02-tech-stack-and-architecture.md) |
| 05 | `@Controller()`, routing, reading params and body | Every REST controller, starting in [06-phase-2-PRD-auth-and-users.md](06-phase-2-PRD-auth-and-users.md) |
| 06 | `@Injectable()`, services, what a provider means | Every service in the project |
| 07 | Dependency injection, constructor injection | Every service and controller constructor, and specifically the Prisma and Mongo model injection covered again below |
| 08 | DTOs and interfaces | A real DTO class per write endpoint, starting in phase 2, replacing the loose `Partial<T>` pattern seen in the MongoDB reference project |
| 09 | REST and HTTP methods | The full REST surface across phases 2 through 6 |
| 10 | Validation and pipes, `ValidationPipe`, custom pipes | The global `ValidationPipe` in [05-phase-1-PRD-foundations-and-config.md](05-phase-1-PRD-foundations-and-config.md), plus one custom pipe in the enrollment flow |
| 11 | Guards, `CanActivate`, `Reflector`, roles | The JWT auth guard and a custom roles guard, in [06-phase-2-PRD-auth-and-users.md](06-phase-2-PRD-auth-and-users.md) |
| 12 | Middleware | The request logging middleware in [05-phase-1-PRD-foundations-and-config.md](05-phase-1-PRD-foundations-and-config.md) |
| 13 | Exception filters | The global exception filter in [05-phase-1-PRD-foundations-and-config.md](05-phase-1-PRD-foundations-and-config.md) |
| 14 | Lifecycle events, `OnModuleInit`, shutdown hooks | The Prisma and Mongoose connection lifecycle hooks, in [05-phase-1-PRD-foundations-and-config.md](05-phase-1-PRD-foundations-and-config.md) |
| 15 | Environment variables and `@nestjs/config` | Startup env validation, in [05-phase-1-PRD-foundations-and-config.md](05-phase-1-PRD-foundations-and-config.md), fixing the exact missing variable problem found in every one of the other seven source folders |
| 16 | MongoDB and NoSQL, conceptually | The notifications module, in [10-phase-6-PRD-notifications-mongodb.md](10-phase-6-PRD-notifications-mongodb.md) |
| 17 | Testing in NestJS | [13-phase-9-PRD-testing-strategy.md](13-phase-9-PRD-testing-strategy.md) |
| 18 | The full request lifecycle | The end to end trace exercise described in [15-roadmap-and-milestones.md](15-roadmap-and-milestones.md) |

## From MongoDB-with-Nest-JS-main

| Note | Concept | Where it lives in LearnBridge |
|---|---|---|
| 02 | `MongooseModule.forRoot`, connecting to MongoDB | [05-phase-1-PRD-foundations-and-config.md](05-phase-1-PRD-foundations-and-config.md) and [10-phase-6-PRD-notifications-mongodb.md](10-phase-6-PRD-notifications-mongodb.md) |
| 03 | `@Schema`, `@Prop`, `SchemaFactory.createForClass` | The `Notification` schema in phase 6 |
| 04 | `MongooseModule.forFeature`, `@InjectModel` | The notifications service in phase 6 |
| 05 | CRUD with a Mongoose model, `find`, `findById`, `findByIdAndUpdate` | Marking a notification as read, and listing a user's notifications, in phase 6 |
| 06 | Embedding a subdocument | The notification's own small embedded metadata object (for example, which course or which review triggered it), in phase 6 |
| 07, 08 | Referencing, one to one, one to many, many to many, `.populate()` | Deliberately not repeated with Mongoose in LearnBridge, because every referencing relationship in this product is instead built relationally with Prisma, see the next table and [04-data-model-and-relationships.md](04-data-model-and-relationships.md) for exactly why that choice was made |
| 09 | Testing gaps, missing providers in spec files | The exact mistake avoided on purpose in [13-phase-9-PRD-testing-strategy.md](13-phase-9-PRD-testing-strategy.md) |
| 10 | Embedding versus referencing tradeoffs | The explicit two database decision explained in [02-tech-stack-and-architecture.md](02-tech-stack-and-architecture.md) and [04-data-model-and-relationships.md](04-data-model-and-relationships.md) |

## From PostgreSQL-with-NEST-JS-main

| Note | Concept | Where it lives in LearnBridge |
|---|---|---|
| 01 | Connecting to Postgres | Done through Prisma instead of TypeORM in the main build, and through TypeORM directly in the optional appendix, [14-phase-10-PRD-typeorm-comparison-appendix.md](14-phase-10-PRD-typeorm-comparison-appendix.md) |
| 02 | Relational concepts, tables, foreign keys | Every entity in [04-data-model-and-relationships.md](04-data-model-and-relationships.md) |
| 03 | TypeORM entities versus Mongoose schemas | Directly repeated as TypeORM versus Prisma versus Mongoose, in the appendix phase |
| 04, 05 | Repository pattern, module CRUD, controller routes | The courses, categories, enrollments, orders, and reviews modules, phases 3 through 5 |
| 06 | A dead module stub with no service or controller | Explicitly avoided, every module in LearnBridge's PRD lists a real controller and service, checked off in [16-definition-of-pro-checklist.md](16-definition-of-pro-checklist.md) |
| 07 | A custom guard verifying an external token, and a guard defined but never actually applied | The JWT guard in phase 2, applied deliberately and checked route by route, avoiding the exact unapplied guard bug found in the reference project |
| 08 | Testing gaps | [13-phase-9-PRD-testing-strategy.md](13-phase-9-PRD-testing-strategy.md) |

## From PrismaORM-NeonDB-NestJS-main

| Note | Concept | Where it lives in LearnBridge |
|---|---|---|
| 02 | What Prisma and Neon actually are | [02-tech-stack-and-architecture.md](02-tech-stack-and-architecture.md) |
| 03 | The `schema.prisma` file, models, generator, datasource | [04-data-model-and-relationships.md](04-data-model-and-relationships.md), written as one real `schema.prisma` covering every relational entity |
| 04 | `PrismaService`, `PrismaModule`, `@Global()` | [05-phase-1-PRD-foundations-and-config.md](05-phase-1-PRD-foundations-and-config.md), built as a genuinely global module this time, unlike the reference project |
| 05, 06, 07 | Prisma backed GraphQL model, resolver, and service calls | [11-phase-7-PRD-graphql-layer.md](11-phase-7-PRD-graphql-layer.md) |
| 08 | The generated `schema.gql` | Same phase, generated from the same `@ObjectType`/`@InputType` decorators |
| 09 | Testing gaps, and a project that could not actually boot due to a bad import path | Avoided on purpose by running the real build and real test suite at the end of every phase, not only at the end of the project, see [15-roadmap-and-milestones.md](15-roadmap-and-milestones.md) |

## From GraphQL-with-NestJS-main

| Note | Concept | Where it lives in LearnBridge |
|---|---|---|
| 01, 02 | GraphQL concepts, `GraphQLModule`, Apollo driver, code first schema | [11-phase-7-PRD-graphql-layer.md](11-phase-7-PRD-graphql-layer.md) |
| 03 | `@ObjectType`, `@Field` | The `Course` and `Review` GraphQL types in phase 7 |
| 04 | `@InputType`, and validation decorators that are inert without a global pipe | The create and update course inputs, wired to the same global `ValidationPipe` from phase 1, closing the exact gap found inert in the reference project |
| 05, 06 | The service layer, resolvers, `@Query`, `@Mutation`, `@Args` | Phase 7 |
| 07 | The generated schema and manual test queries | Phase 7, plus a `queries.http` or `.graphql` scratch file kept alongside the module |
| 08 | Testing gaps | [13-phase-9-PRD-testing-strategy.md](13-phase-9-PRD-testing-strategy.md) |

## From JWT-with-Nest-JS-main

| Note | Concept | Where it lives in LearnBridge |
|---|---|---|
| 02 | JWT structure, header, payload, signature, signing and verification | [06-phase-2-PRD-auth-and-users.md](06-phase-2-PRD-auth-and-users.md) |
| 03 | Authentication versus authorization | The auth guard versus the roles guard, both in phase 2, are the concrete answer to this distinction |
| 04 | What a real implementation would need | Everything in phase 2 is that sketch, actually built |

## From JWT-Auth-with-Mongo-DB-Nest-JS-main

| Note | Concept | Where it lives in LearnBridge |
|---|---|---|
| 01, 02 | Connecting auth to a database, the user schema, password handling | Phase 2, rebuilt against a Prisma `User` table instead of a Mongoose schema, same bcrypt hashing discipline |
| 03 | Registration and login flow, and the bug where a failed login returned `null` instead of a 401 | Phase 2, deliberately throwing `UnauthorizedException` on bad credentials and `ConflictException` on a duplicate email, fixing both bugs found in the reference project |
| 04 | JWT signing and the token payload | Phase 2 |
| 05 | The passport JWT strategy and protected routes | Phase 2 |
| 06 | Auth controller routes | Phase 2 |
| 07 | Testing gaps | [13-phase-9-PRD-testing-strategy.md](13-phase-9-PRD-testing-strategy.md) |

## From Rate-Limit-in-NestJS-using-Throttler-main

| Note | Concept | Where it lives in LearnBridge |
|---|---|---|
| 02 | What rate limiting is, `ThrottlerModule.forRoot`, named throttlers | [12-phase-8-PRD-rate-limiting-and-hardening.md](12-phase-8-PRD-rate-limiting-and-hardening.md) |
| 03 | The `APP_GUARD` token, applying a guard globally | Same phase, and the same technique already used for the JWT guard's sibling pattern in phase 2 |
| 04 | Per route `@Throttle()` overrides, and the bug where the override matched the global default and did nothing | Same phase, this time the login and signup routes get a genuinely stricter limit than the global default, so the override actually does something |

## What this table proves

Every single note file across all eight folders has a row above. Nothing was left out, and nothing in LearnBridge was invented just to pad the list, each feature exists because the product genuinely needs it. That is the actual point of building one real project instead of eight small ones, every concept earns its place instead of being demoed in isolation.
