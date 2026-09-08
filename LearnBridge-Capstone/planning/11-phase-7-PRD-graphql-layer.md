# Phase 7 PRD: The GraphQL Layer

## Goal

Expose the course catalog and its reviews through GraphQL, backed by the exact same Prisma data the REST API already uses, so you experience firsthand how the same underlying service and database layer can sit under two completely different API shapes.

## Concepts practiced

`GraphQLModule` configuration with the Apollo driver and code first `autoSchemaFile`, `@ObjectType` and `@Field`, `@InputType`, `@Resolver`, `@Query`, `@Mutation`, `@Args`, and making sure `class-validator` decorators on GraphQL input types are actually enforced, closing the exact gap left inert in `GraphQL-with-NestJS-main`.

## Scope

Register `GraphQLModule.forRoot(ApolloDriverModule, { autoSchemaFile: 'src/schema.gql', sortSchema: true })` in `AppModule`. Build a `CourseType` and a `ReviewType` with `@ObjectType()` and `@Field()`, mirroring the Prisma models but only exposing the fields that make sense to a public API consumer (for instance, never exposing an instructor's raw id if a nested instructor name field is more useful, a real decision worth making deliberately rather than mirroring the database schema one to one out of habit). Build `CreateCourseInput` and `UpdateCourseInput` with `@InputType()`, carrying the same `class-validator` decorators as their REST DTO counterparts, and confirm, by deliberately sending an invalid mutation, that the global `ValidationPipe` registered in phase 1 actually rejects it, since a global pipe registered only for HTTP contexts does not automatically apply to GraphQL resolvers in every Nest version, this needs to be checked against your actual installed version rather than assumed, and if it does not apply automatically, the resolver method itself needs `@UsePipes(new ValidationPipe())` added explicitly.

Build a `CoursesResolver` with a `courses` query (supporting the same pagination and category filtering as the REST endpoint, calling the exact same `CoursesService` method underneath, not a duplicate implementation), a `course` query by id, and a `createCourse` mutation, guarded the same way the REST `POST /courses` route is, with `@UseGuards(JwtAuthGuard)` and a role check, proving that guards work identically in a GraphQL resolver as they do in an HTTP controller. Add a `reviews` field resolver on `CourseType` that resolves a course's reviews on demand, only when a client's query actually asks for that field, the core selling point of GraphQL worth experiencing directly rather than only reading about.

## API surface

The `courses`, `course`, and `reviews` queries, and the `createCourse` mutation, reachable at the single `/graphql` endpoint, explorable through Apollo's sandbox in development the same way `queriesForTesting` was used in the reference project.

## Acceptance criteria

1. Querying `courses` with a selection set that does not include `reviews` never triggers a reviews lookup at all, verified by temporarily adding a log line inside the reviews field resolver and confirming it does not fire for that request.
2. Sending a `createCourse` mutation with a blank title is rejected with a clear GraphQL error, not silently accepted, proving the input validation actually runs.
3. Calling `createCourse` with a valid STUDENT token, rather than an INSTRUCTOR token, is rejected the same way the equivalent REST route rejects it.
4. The REST `GET /courses/:id` and the GraphQL `course` query, given the same id, return data that agrees with each other, since they are reading through the same service and the same database.

## Explicit trap to avoid

Do not write a second, parallel implementation of course lookup logic inside the resolver just because it is convenient to inline a Prisma call there. The resolver should call `CoursesService`, the exact same service the REST controller calls, so there is only ever one real implementation of "how a course gets fetched" in the entire codebase.
