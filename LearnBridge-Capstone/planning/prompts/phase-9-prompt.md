# Phase 9 Prompt: Testing Strategy

Use this once phase 8 is complete and the route audit shows no unprotected routes that should be protected.

Copy everything inside the fenced block below into Claude Code as one message.

```
Before writing any code, read planning/13-phase-9-PRD-testing-strategy.md completely.

Go through every service and controller in the entire project, from phase 1 through phase 8, and list, explicitly, which ones currently only have a placeholder or partial test and which have none at all. For every service that depends on PrismaService, ensure its test file provides a hand built mock shaped like the real Prisma client for exactly the model methods that service actually calls, for example a mocked course object with jest.fn() implementations for findMany, findUnique, create, and update, never a bare, empty providers array for any class whose constructor actually needs something injected. For NotificationsService and any other Mongoose backed class, provide a mock using getModelToken from @nestjs/mongoose with a useValue object implementing only the methods actually called. For every controller, provide a fully mocked version of its service the same way, do not register the bare controller alone.

Specifically write, if they do not already exist from an earlier phase, or expand if they do, two sets of business logic tests that matter more than the rest. First, for OrdersService, tests using a mocked $transaction that actually invokes its callback, covering both the forced success branch, asserting an enrollment gets created and the order becomes PAID, and the forced failure branch, asserting the order becomes FAILED and asserting, with an explicit call count check, that enrollment creation was never invoked at all. Second, for ReviewsService, a test asserting a review attempt with no matching enrollment throws ForbiddenException, and a second asserting one with a matching enrollment succeeds.

Set up a genuinely separate end to end test environment. Create a .env.test file, or an equivalent mechanism, defining TEST_DATABASE_URL and TEST_MONGO_URI pointing at disposable databases that are never the same ones the app uses in normal local development, and wire the e2e Jest config to load that file. Write a global setup step for the e2e suite that resets the test Postgres database to a clean migrated state and clears the test MongoDB notifications collection before the suite runs.

Write one full end to end test covering signup, then login, then a call to GET /users/me with the resulting token, asserting each step's status code and that the final profile response contains no passwordHash field. Write a second full end to end test covering the complete order to enrollment to notification chain, creating a real instructor and course first, then a real student, placing a real order against the real test database with the payment simulation forced to succeed for this specific test, and asserting, at the end, that the order is PAID, the enrollment exists, and a notification for the instructor exists in the test MongoDB database.

Run the entire unit and end to end suite from a clean state twice in a row, and confirm both runs produce identical results, if they do not, find and fix whatever test is depending on leftover state from a previous run before considering this phase done.

When you are done, give me the full test output summary, and tell me explicitly how many tests exist now compared to how many existed at the start of this phase, and name the one test you are most confident would have caught a real bug if it had existed during an earlier phase.
```
