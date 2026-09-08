# Phase 5 PRD: Orders and Payments

## Goal

Make enrollment happen only as the result of a paid order, and make that pairing atomic, so that a failure partway through can never leave the system in a state where money moved with no enrollment to show for it, or an enrollment exists that nothing ever actually paid for.

## Concepts practiced

Database transactions, specifically Prisma's `$transaction`, one to one relationships, and simulating an external, unreliable dependency (a payment provider) in a way that still forces you to handle its failure correctly.

## Scope

An `OrdersModule`. `POST /orders` accepts a courseId from an authenticated student, and does the following inside a single `prisma.$transaction`: creates an `Order` row with status PENDING and the course's current price, simulates calling a payment provider (a small function that succeeds most of the time and fails deliberately and randomly some small percentage of the time, specifically so you are forced to write and test the failure branch, not just the happy path), and only if that simulated payment succeeds, updates the order's status to PAID and creates the corresponding `Enrollment` row, then sets the order's `enrollmentId` to point at it. If the simulated payment fails, the transaction updates the order's status to FAILED and rolls back, no `Enrollment` row ever gets created, and the transaction as a whole must guarantee that either both the PAID order and the Enrollment exist, or neither of them does, there is no valid third state.

The existing standalone enrollment creation endpoint from phase 4 gets restricted at this point to admin only, or removed entirely, since in the finished product, a normal student can no longer create an enrollment except through this order flow, this is a deliberate, visible change from phase 4's PRD, worth noting explicitly in your own build log when you get there.

## API surface

`POST /orders`, student facing. `GET /orders/mine`, listing a student's own order history with its status. `GET /orders/:id`, returning one order, restricted to the student who placed it or an admin.

## Acceptance criteria

1. A successful order results in exactly one PAID order and exactly one Enrollment row, linked to each other by id in both directions.
2. Forcing the simulated payment failure path, either by running it enough times to hit the random failure or by temporarily hardcoding it to always fail during testing, results in exactly one FAILED order and zero Enrollment rows, never a partial state.
3. Attempting to order the same course twice while a PAID order and Enrollment already exist for it is rejected with a clear conflict response before any payment simulation even runs.
4. Killing the database connection mid transaction (a manual test worth actually trying once, not just reasoning about) never leaves a PAID order without a matching Enrollment.

## Explicit trap to avoid

Do not create the Order and the Enrollment as two separate, sequential service calls outside of a shared transaction. Two separate calls means two separate chances to succeed or fail independently of each other, which reopens exactly the partial state problem this whole phase exists to close. Everything that must happen together needs to be inside one `$transaction` call.
