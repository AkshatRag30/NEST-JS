# Phase 7 Prompt: The GraphQL Layer

Use this once phase 6 is complete and notifications are visibly appearing after a real enrollment.

Copy everything inside the fenced block below into Claude Code as one message.

```
Before writing any code, read these files completely: planning/11-phase-7-PRD-graphql-layer.md, and the Course and Review sections of planning/04-data-model-and-relationships.md. This is phase 7.

Before picking dependency versions yourself, look at the exact working versions already listed in ../GraphQL-with-NestJS-main/package.json and ../PrismaORM-NeonDB-NestJS-main/package.json, two sibling reference projects one directory up from this project, and match those versions for @nestjs/graphql, @nestjs/apollo, and graphql rather than guessing at whatever the latest tag happens to be, since those exact versions are already proven to work together with this version of NestJS.

Register GraphQLModule in AppModule using the ApolloDriver, with autoSchemaFile pointed at src/schema.gql and sortSchema enabled, following whatever the exact current API shape for GraphQLModule.forRoot is in the version you installed, check the installed package's own type definitions if the exact option names are unclear rather than guessing.

Build a CourseType and a ReviewType using @ObjectType and @Field, deliberately choosing not to mirror the Prisma schema field for field, expose id, title, description, price, published, and createdAt on CourseType, plus a nested instructor field exposing only id and name, and a reviews field that is resolved lazily rather than being a plain eagerly loaded array.

For the create and update course inputs, first try decorating the exact same DTO classes already built in phase 3 with both class-validator decorators and @InputType and @Field, so there is only one real class describing what a valid course creation payload looks like, used by both the REST controller and the GraphQL resolver. If this causes real friction because of how the two decorator systems interact, fall back to writing separate CreateCourseInput and UpdateCourseInput classes instead, but only after actually trying the shared approach first, and tell me at the end which path you took and why.

Build a CoursesResolver with a courses query supporting the same pagination and category filtering as the REST GET /courses endpoint, a course query by id, and a createCourse mutation guarded exactly the same way the REST POST /courses route is, reusing role checking guard, every one of these resolver methods must call the exact same CoursesService methods the REST controller already calls, never a second, separate implementation of the same lookup or creation logic. Add a reviews field resolver on CourseType using @ResolveField, calling ReviewsService only when a query actually asks for that field, add a temporary console log inside it, send a courses query that does not select reviews, confirm the log does not fire, and send one that does select it, confirm the log does fire, then tell me what you observed before removing the temporary log.

Test, by actually sending a malformed createCourse mutation with a blank title through the GraphQL endpoint, whether the class-validator decorators on your input type are enforced automatically by the existing global ValidationPipe from phase 1, or whether GraphQL resolver arguments in the installed version need an explicit @UsePipes(new ValidationPipe()) added to the resolver method to be validated. Do not assume either way, show me the actual result of that test, and add the explicit pipe if the global one does not cover it.

When you are done, give me one example GraphQL query and one example mutation I can paste into the Apollo sandbox, and confirm that querying a course by id through GraphQL and through the REST GET /courses/:id endpoint, for the same course, return data that agrees with each other.
```
