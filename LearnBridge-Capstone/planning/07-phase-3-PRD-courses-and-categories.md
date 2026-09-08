# Phase 3 PRD: Courses and Categories

## Goal

Build the first real one to many relationship (instructor to course) and the first real many to many relationship (course to category), both enforced relationally through Prisma, and build the first public, unauthenticated browsing endpoints in the product.

## Concepts practiced

Prisma model relations (`@relation`), the repository style calls (`create`, `findMany`, `update` with `where` and `include`) already seen in `PrismaORM-NeonDB-NestJS-main`, DTOs with nested validation, pagination and filtering query parameters, and applying the guards built in phase 2 to real routes for the first time.

## Scope

A `CoursesModule` with create, update, publish or unpublish, and delete, every write route protected by `JwtAuthGuard` and `@Roles('INSTRUCTOR')`, and every write route additionally checking that the course being modified actually belongs to the currently authenticated instructor, throwing `ForbiddenException` if an instructor tries to edit someone else's course, which is a business rule no guard alone can express, it has to be checked in the service against `req.user.id`. A `CategoriesModule`, simpler, with create and list, restricted to `ADMIN` for writes and open to everyone for reads.

Creating or updating a course accepts a list of category ids, and the service is responsible for writing the corresponding rows into the `CourseCategory` join table, this is the one place in the relational half of the product where you manage a many to many relationship by hand rather than Prisma doing it invisibly for you, giving you a direct, concrete comparison against the array based many to many pattern from the `Project`/`Developer` reference example.

Browsing is public. `GET /courses` supports pagination (`page` and `limit` query parameters, validated and defaulted sensibly) and filtering by category id, and only ever returns published courses to an unauthenticated caller, while an authenticated instructor calling a separate `GET /courses/mine` route sees their own courses regardless of published state. `GET /courses/:id` returns one course with its categories included, using Prisma's `include`, the direct equivalent of the `.populate()` calls you already used repeatedly in the MongoDB reference project.

## API surface

`POST /courses`, `PATCH /courses/:id`, `PATCH /courses/:id/publish`, `DELETE /courses/:id`, all instructor only and ownership checked. `GET /courses`, `GET /courses/:id`, both public. `GET /courses/mine`, instructor only, no ownership check needed since it only ever returns the caller's own data by construction. `POST /categories`, `GET /categories`, admin write, public read.

## Acceptance criteria

1. An instructor can create a course with two category ids and immediately fetch it back with both categories present in the response.
2. A second instructor attempting to update the first instructor's course receives a 403, even though their token is otherwise perfectly valid.
3. An unpublished course never appears in the public `GET /courses` list, but does appear in the owning instructor's `GET /courses/mine` list.
4. Filtering `GET /courses?categoryId=...` returns only courses actually linked to that category through the join table, not every course.
5. Pagination parameters that are missing default sensibly, and pagination parameters that are invalid (for example a negative page number) are rejected by the DTO validation from phase 1, not silently clamped.

## Explicit trap to avoid

Do not let the ownership check happen only on the frontend or only be implied by which routes exist. The service method itself must fetch the course, compare its `instructorId` against the caller, and throw before doing any write, every single time, because a guard checking role alone would let any instructor edit any course.
