# Phase 5 Prompt: Orders and Payments

Use this once phase 4 is complete and you have personally enrolled a student in a course through the standalone endpoint that phase built.

Copy everything inside the fenced block below into Claude Code as one message.

```
Before writing any code, read these files completely: planning/04-data-model-and-relationships.md (the Order section) and planning/09-phase-5-PRD-orders-and-payments.md. This is phase 5. Read the following clarification carefully before implementing anything, since it corrects a slightly loose sentence in the PRD itself.

The transaction in this phase has exactly two valid outcomes, never a third. Outcome one, the simulated payment succeeds, in which case, inside a single prisma.$transaction call, the order is created with status PAID (or created PENDING and then updated to PAID within the same transaction, either is fine), a matching Enrollment row is created, and the order's enrollmentId is set to point at it, and all of that commits together. Outcome two, the simulated payment fails, in which case, still inside that same transaction, an Order row is created and committed with status FAILED, recording that the attempt happened, but no Enrollment row is created at all, this is a deliberate, successful commit of a failed order, it is not a rollback in the sense of undoing anything, the FAILED order is real, permanent, and correct data. A true rollback, where nothing at all persists, not even a FAILED order, only happens if something unexpected goes wrong, like losing the database connection mid transaction, which Prisma's transaction mechanism already handles for you by rolling back automatically, you do not need to write special handling for that case beyond letting the error propagate. Do not implement a version where a failed payment causes the entire transaction, including the FAILED order record itself, to disappear, that would be a real bug, not the intended behavior.

Add an Order model to prisma/schema.prisma with id, a studentId foreign key, a courseId foreign key, a nullable and unique enrollmentId foreign key to Enrollment, amount using Decimal, a status field using a Prisma enum with values PENDING, PAID, and FAILED, and createdAt. Migrate.

Write a small, isolated simulatePayment function, in its own file, that returns a boolean, succeeding roughly eighty percent of the time at random by default, but accepting an optional forced outcome parameter so it can be deterministically forced to succeed or fail in tests, do not make tests depend on randomness.

Build an OrdersModule. Its create method takes the authenticated student's id and a courseId, and before doing anything else, checks whether a PAID order already exists for that exact student and course, returning a ConflictException immediately if so, without ever calling the payment simulation. Otherwise, it runs the two outcome transaction exactly as clarified above, fetching the course's current price to set as the order's amount. Add a findMine method and a findOne method, the latter restricted to the order's own student or an ADMIN.

Go back into EnrollmentsController from phase 4 and restrict POST /enrollments to the ADMIN role only, since a normal student can no longer create an enrollment directly, only through a successful order, add a short comment at that route explaining why it changed, referencing this phase.

Build an OrdersController exposing POST /orders, GET /orders/mine, and GET /orders/:id.

Write focused unit tests for OrdersService.create using a mocked PrismaService whose $transaction implementation actually invokes the callback function it is given, passing a mocked transaction client. First test, forcing simulatePayment to succeed, assert both an order update to PAID and an enrollment creation call happened. Second test, forcing simulatePayment to fail, assert the order was updated to FAILED and assert, explicitly, that no enrollment creation call happened at all. Third test, attempting to order a course a second time after a PAID order already exists, assert a ConflictException is thrown and that simulatePayment was never called.

When you are done, tell me, from your own testing, what the database actually contains after a forced payment failure, specifically confirm there is a FAILED order row and zero enrollment rows for that attempt, and explain in your own words why that is correct and not a bug.
```
