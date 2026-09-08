# Phase 4 Prompt: Enrollments and Reviews

Use this once phase 3 is complete and you have a real published course sitting in your database with at least one category attached.

Copy everything inside the fenced block below into Claude Code as one message.

```
Before writing any code, read these files completely: planning/04-data-model-and-relationships.md (the Enrollment and Review sections) and planning/08-phase-4-PRD-enrollments-and-reviews.md. This is phase 4. Do not build orders or notifications yet, and be aware, explicitly, that this phase's enrollment creation endpoint will be restricted in the very next phase once payments exist, that is expected and intentional, not a mistake to avoid now.

Add an Enrollment model to prisma/schema.prisma with id, a studentId foreign key to User, a courseId foreign key to Course, enrolledAt defaulting to now, progressPercent as an integer defaulting to 0, a completed boolean defaulting to false, and a database level unique constraint across the combination of studentId and courseId, so a student physically cannot enroll in the same course twice even if two requests race each other, application level checks alone are not enough here. Add a Review model with id, rating as an integer, comment, a userId foreign key, a courseId foreign key, and createdAt. Migrate.

Build an EnrollmentsModule. Its create method takes the authenticated student's id and a courseId, and attempts the insert directly, catching Prisma's unique constraint violation error code (P2002) and converting it into a ConflictException with a clear message, rather than checking for existence first and then inserting as two separate steps, which would still leave a race condition. It needs a findMine method scoped to the caller. It needs an updateProgress method restricted to the enrollment's own student, which sets progressPercent and automatically sets completed to true once progress reaches 100 or higher, in the same update. Write a custom pipe, a class implementing PipeTransform, that validates an incoming progress value is a number between 0 and 100 inclusive before it ever reaches the controller method's body, rejecting anything outside that range with a BadRequestException, and apply it to the relevant route parameter or body field.

Build an EnrollmentsController exposing POST /enrollments (student role, temporary for this phase only), GET /enrollments/mine, and PATCH /enrollments/:id/progress, all requiring authentication, the progress route additionally checking the enrollment being updated actually belongs to the caller.

Build a ReviewsModule. Its create method takes the authenticated user's id, a courseId, a rating, and a comment, and before creating anything, queries for an Enrollment matching that studentId and courseId, throwing a ForbiddenException with a clear message if none exists, only after that check passes does it create the review. It needs a findByCourse method. In the CoursesService's findOne method from phase 3, add a genuinely computed average rating using Prisma's aggregate query with an avg over the rating column grouped by courseId, merged into the course response, not a stored or manually recalculated field.

Build a ReviewsController exposing POST /reviews (any authenticated user, the enrollment check inside the service is what actually gates it, not a role) and GET /courses/:id/reviews (public).

Write two focused unit tests. First, that EnrollmentsService.create called twice with the same student and course results in a ConflictException the second time, using a mocked PrismaService that simulates the P2002 error on the second call. Second, that ReviewsService.create throws a ForbiddenException when no matching enrollment is found, and succeeds when one is, both with a mocked PrismaService.

When you are done, tell me what happens, step by step, when a student who has never enrolled in a course tries to submit a review for it, and confirm the average rating on a course actually changes after you submit a new review through the real running app.
```
