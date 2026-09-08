# Phase 9 PRD: Testing Strategy

## Goal

Write a test suite that actually passes, fixing, deliberately and by name, the single most repeated failure found across every one of the eight reference projects, a `TestingModule` built without supplying every constructor dependency a class actually needs.

## Concepts practiced

Unit testing with `@nestjs/testing`, mocking a Prisma client and a Mongoose model with `jest.fn()` stand ins, using `getModelToken` for Mongoose and a manually provided token or a full mock service for Prisma dependent services, and end to end testing against a real, disposable test database.

## Scope

For every service written across phases 1 through 8, write a unit test that provides a hand written mock for every constructor dependency, never a bare, empty `providers: [SomeService]` array when that service actually needs something injected. For a Prisma backed service, this means providing an object shaped like `{ provide: PrismaService, useValue: { course: { findMany: jest.fn(), create: jest.fn(), ... } } }`, exactly the kind of stand in described but never actually written in any of the eight reference projects' spec files. For the `NotificationsService`, this means providing `{ provide: getModelToken(Notification.name), useValue: { find: jest.fn(), ... } }`. For every controller, provide a mocked version of its service the same way, rather than the bare controller only registration that failed across nearly every reference project's controller spec files.

Write true business logic tests, not just "should be defined" placeholders, specifically for the two most important rules in the product, the order transaction from phase 5 (a mocked Prisma `$transaction` call that simulates both the success and the deliberate failure branch, asserting that a failure never results in a call to create an `Enrollment`) and the review eligibility check from phase 4 (asserting a review attempt without an enrollment is rejected, and one with an enrollment succeeds).

Set up a separate end to end test environment, a second `DATABASE_URL` and `MONGO_URI` pointed at disposable test databases, never the same ones the app runs against in development, with a setup step that clears and reseeds them before each test run. Write end to end tests for the full signup, login, and one protected route flow, since that sequence is the backbone every other feature depends on, and for the full order to enrollment to notification chain from phases 5 and 6, since that sequence is the one place in the product where a bug would be both easy to introduce and expensive to have in a real product.

## Acceptance criteria

1. Running the full unit test suite passes with zero tests failing due to a missing dependency injection provider, verified by deliberately checking that every `Test.createTestingModule` call either provides every real dependency or a real mock for it.
2. The order transaction unit test genuinely exercises both branches, a run where the simulated payment succeeds and a run where it is forced to fail, with a distinct assertion for each.
3. The end to end signup, login, protected route sequence passes against a real, running test database, not a mocked one, since the entire point of an end to end test is to catch a wiring mistake a unit test's mocks would hide.
4. Deleting the test database between runs and re running the suite produces the exact same results, proving no test depends on leftover state from a previous run.

## Explicit trap to avoid

Do not write a unit test for any class with constructor dependencies by only listing that one class in `providers` or `controllers` and hoping `Test.createTestingModule(...).compile()` will figure the rest out. It will not, and this exact mistake is why the majority of spec files across the eight reference projects fail the instant you actually run them rather than only reading them.
