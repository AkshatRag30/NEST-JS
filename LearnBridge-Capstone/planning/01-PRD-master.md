# Master PRD: LearnBridge

## Summary

LearnBridge is a course marketplace API. Instructors publish courses, students browse and enroll in them, pay for them, leave reviews once enrolled, and receive notifications about activity on their account. It is deliberately shaped like a real, small SaaS product rather than a single toy resource, because that shape is what forces every concept from the eight source folders to show up naturally instead of being bolted on.

## Problem statement

You already have detailed conceptual notes on NestJS fundamentals, MongoDB, PostgreSQL, Prisma, GraphQL, JWT authentication, and rate limiting. Notes are not the same skill as building. The gap between reading how `@InjectModel` or `@Prop({ type: [Tag] })` works and being able to sit down and design, build, break, and fix a real feature using that idea is exactly the gap this project closes.

## Goals

The finished product must let a visitor register an account as either a student or an instructor. It must let an instructor create, update, and publish courses, organized under one or more categories. It must let a student browse and search published courses, enroll in one by paying for it, track their own progress through it, and leave a review once enrolled. It must record account activity, such as a new enrollment or a new review on your course, as notifications a user can fetch and mark as read. It must expose the course catalog through both a REST API and a GraphQL API. It must protect every account level action with real JWT authentication, and protect the login and signup endpoints specifically with rate limiting.

## Non goals

This project is not trying to teach you payment gateway integration, so the payment step is a simulated internal ledger entry, not a real Stripe or PayPal integration, though it is built with the same discipline (an amount, a status, a timestamp, and a transaction that must not half complete) that a real integration would need. It is not trying to teach you a frontend framework, there is no UI here, only the API, verified with tools like Postman, Insomnia, or the GraphQL sandbox. It is not trying to teach you deployment or infrastructure, containerization and cloud hosting are worth learning but are a separate project.

## Target users, as personas

A student is someone who creates an account, searches the catalog, enrolls in courses, tracks progress, and leaves reviews. An instructor is someone who creates an account, publishes and manages their own courses, and can see reviews and enrollment counts for what they teach. An admin is someone who can manage categories and see everything, mainly included so the project has a real reason to build role based authorization instead of a single flat permission level.

## Full feature list

1. Account registration and login with a hashed password and a signed JWT, separately for the student and instructor roles, with an admin role that exists but is seeded rather than self registered.
2. Role based route protection, so an instructor only endpoint genuinely rejects a student token, not just in theory.
3. Category management, with a many to many relationship between courses and categories.
4. Course management, with a one to many relationship from instructor to courses, including create, update, publish and unpublish, and delete.
5. Course browsing and search for anyone, authenticated or not, with pagination and filtering by category.
6. Enrollment, which is a many to many relationship between students and courses that carries its own data, an enrollment date, a progress percentage, and a completed flag, so it has to be modeled as its own entity rather than a bare join array.
7. A simulated payment step that creates an order and, only if that order succeeds, creates the enrollment, inside one database transaction, so a failed payment can never leave an enrollment behind with nothing paid for it.
8. Reviews, restricted to students who are actually enrolled in the course they are reviewing, which is a one to many relationship from both the user and the course into the review.
9. Notifications, stored in MongoDB rather than PostgreSQL on purpose, created whenever a meaningful event happens (a new enrollment, a new review on your course), fetchable by the user they belong to, and markable as read.
10. A GraphQL query and mutation surface mirroring the course catalog and reviews, built on the same Prisma backed data as the REST API, not a separate copy of it.
11. Global rate limiting on every route, with a stricter limit specifically on login and signup to blunt brute force attempts.
12. A global validation pipe, a global exception filter, and a request logging middleware, applied once, application wide, not bolted onto individual routes by hand.
13. Environment variable validation at startup, so a missing `DATABASE_URL`, `MONGO_URI`, or `JWT_SECRET` fails loudly the moment the app tries to boot, with a clear message, instead of failing confusingly later or silently connecting to nothing, which is the exact failure every single one of the eight reference projects was left exposed to.
14. A real test suite, unit tests with every dependency properly mocked and end to end tests against a real test database, specifically built to avoid the missing provider mistake that broke nearly every spec file across the eight reference projects.

## Success criteria

You will know this project succeeded, not when it runs, but when you can explain, out loud, without opening the code, why enrollment needed its own entity instead of a plain array of ids, why the payment step needed a transaction, why notifications live in a different database than everything else, and why the environment variable validation step exists at all. Those four questions are a fair test of whether the underlying concepts actually landed, or whether you only copied a pattern.
