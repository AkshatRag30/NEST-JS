# Phase 3 Prompt: Courses and Categories

Use this once phase 2 is complete and you have personally signed up and logged in as at least one instructor and one student through the real running app.

Copy everything inside the fenced block below into Claude Code as one message.

```
Before writing any code, read these files completely: planning/04-data-model-and-relationships.md (the Category and Course sections) and planning/07-phase-3-PRD-courses-and-categories.md. This is phase 3, building on the working auth system from phase 2. Do not touch enrollments, reviews, orders, or notifications yet.

Add a Category model (id, a unique name) and a Course model (id, title, description, price using the Decimal type since this is money, a published boolean defaulting to false, an instructorId foreign key relation to User, createdAt, and updatedAt using Prisma's @updatedAt) to prisma/schema.prisma. Add a CourseCategory join model with a composite primary key on courseId and categoryId, and foreign key relations from each to Course and Category respectively. Migrate the database.

Build a CategoriesModule. Its service supports creating a category and listing all of them. Its controller exposes POST /categories, protected by the JWT guard and the roles guard restricted to ADMIN, and GET /categories, completely public, no guard at all.

Build a CoursesModule. Write a CreateCourseDto (title, description, price, and a categoryIds array of identifiers, all properly validated with class-validator) and an UpdateCourseDto that makes every one of those fields optional, using PartialType from @nestjs/mapped-types rather than writing a second full DTO by hand.

The CoursesService needs a create method that takes the authenticated instructor's id and the DTO, creates the course, and writes the corresponding rows into the CourseCategory join table for every category id supplied, using a nested Prisma write in the same call rather than a separate loop of individual inserts if the API allows it cleanly, otherwise a clear loop is fine. It needs an update method that first fetches the course, checks that its instructorId actually matches the caller's id, and throws a ForbiddenException if it does not, before making any change, then updates the changed fields and reconciles the category links, adding new ones and removing ones no longer present in the DTO. It needs a publish and an unpublish method, or a single method that flips the published boolean, with the exact same ownership check. It needs a remove method with the same ownership check. It needs a findPublished method supporting page and limit query parameters with sane validated defaults, and an optional categoryId filter, that only ever returns courses where published is true. It needs a findMine method, scoped to the caller's own instructorId regardless of published state. It needs a findOne method returning a single course with its categories and a minimal instructor summary (id and name only) included through Prisma's include option.

Build a CoursesController exposing POST /courses, PATCH /courses/:id, PATCH /courses/:id/publish, and DELETE /courses/:id, all guarded for the INSTRUCTOR role, and GET /courses, GET /courses/:id, both public, and GET /courses/mine, guarded for the INSTRUCTOR role with no ownership check needed since it only returns the caller's own data by construction.

Write one focused unit test proving the ownership check actually works, a second instructor's id attempting to update the first instructor's course must result in a ForbiddenException being thrown, mock PrismaService by hand for this test.

When you are done, tell me the exact request body for creating a course with two category ids, and confirm out loud what happens, step by step, when a second instructor tries to edit a course they do not own.
```
