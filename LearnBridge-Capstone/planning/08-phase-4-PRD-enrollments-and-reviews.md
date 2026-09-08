# Phase 4 PRD: Enrollments and Reviews

## Goal

Build the many to many relationship that actually carries its own data, the single most important modeling idea this whole project exists to teach, and build a review system whose write permission depends on a completely different relationship already existing.

## Concepts practiced

Modeling a many to many relationship as its own first class entity rather than a bare join, custom validation logic that depends on more than one table, and a custom pipe. See [04-data-model-and-relationships.md](04-data-model-and-relationships.md) for the full reasoning behind why Enrollment needed its own entity.

## Scope

An `EnrollmentsModule`. For this phase, before payments exist in phase 5, enrollment can be created directly by a student hitting an endpoint (phase 5 will change this so enrollment only ever happens as the result of a successful order, but building it standalone first lets you get the entity and its rules right before adding the transactional complexity on top). Creating an enrollment checks that one does not already exist for that student and course, throwing `ConflictException` if it does, a student cannot enroll in the same course twice. A `PATCH /enrollments/:id/progress` endpoint lets a student update their own `progressPercent`, and automatically sets `completed` to true once progress reaches 100, this is a good place to write one small custom pipe that clamps or rejects an out of range progress value before it ever reaches the service.

A `ReviewsModule`. Creating a review requires the caller to actually have an `Enrollment` row for that course, checked in the service with a real database query, not assumed, throwing `ForbiddenException` if no enrollment exists, this is the concrete business rule where one relationship (Enrollment) gates another (Review). A course's average rating, exposed on `GET /courses/:id`, is computed with a Prisma aggregate query (`avg` over the `rating` column grouped by `courseId`) rather than fetched and averaged manually in application code, giving you a real reason to learn Prisma's aggregation API rather than only its basic CRUD calls.

## API surface

`POST /enrollments`, `GET /enrollments/mine`, `PATCH /enrollments/:id/progress`, all student facing and ownership checked. `POST /reviews`, `GET /courses/:id/reviews`, the first public, the second requiring the enrollment check described above.

## Acceptance criteria

1. A student can enroll in a course exactly once, a second attempt returns a 409.
2. A student who has never enrolled in a course receives a 403 when attempting to review it, even with an otherwise perfectly valid token and a well formed request body.
3. Setting `progressPercent` to 100 through the progress endpoint automatically flips `completed` to true in the same request, verified by immediately fetching the enrollment back.
4. `GET /courses/:id` includes a genuinely computed average rating that changes correctly after a new review is submitted, not a stale or hardcoded value.

## Explicit trap to avoid

Do not model Enrollment as a plain array of course ids sitting on the User, or a plain array of student ids sitting on the Course, the way the many to many example in the MongoDB reference project did. The moment you find yourself wanting to attach a date, a percentage, or a boolean to a relationship, that relationship needs its own table with its own primary key, not a shortcut array on either side.
