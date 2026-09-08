# Data Model and Relationships

## Why almost everything lives in PostgreSQL

Every relationship in this product except notifications benefits from being enforced by the database itself. A course must belong to a real instructor, an enrollment must point at a real student and a real course, an order must point at a real enrollment, and none of that should ever be allowed to point at something that does not exist. That is exactly what a foreign key in a relational database is for, and it is the reason the MongoDB reference project's referencing pattern, an `ObjectId` with a `ref`, has to be kept consistent entirely by application code with no database level guarantee behind it. LearnBridge uses that easier, unenforced style only for the one piece of data where it is actually the right tradeoff, notifications, covered at the end of this file.

## User

Fields: id, email (unique), passwordHash, name, role (an enum of STUDENT, INSTRUCTOR, or ADMIN), createdAt. A user is never stored with a plain text password anywhere, only the bcrypt hash, matching the discipline already used correctly in `JWT-Auth-with-Mongo-DB-Nest-JS-main`, and the role field is what makes real role based guards meaningful instead of decorative.

## Category

Fields: id, name (unique). Deliberately the simplest entity in the system, its entire purpose is to sit on the other side of a many to many relationship with Course.

## Course

Fields: id, title, description, price, published (boolean), instructorId (foreign key to User), createdAt, updatedAt. The relationship from User to Course, specifically from an instructor to their courses, is one to many, one instructor can write many courses, and a course has exactly one instructor. This is the same shape as the `Library` to `Book` relationship you already saw, just enforced relationally with a foreign key column instead of an array of ObjectIds.

Course also relates to Category through an explicit join table, CourseCategory, with courseId and categoryId as a composite key. This is a many to many relationship, a course can sit under several categories and a category groups several courses, the same shape as the `Project` to `Developer` relationship in the MongoDB reference project, but built the relational way, one join table instead of an array of ids kept in sync on both sides by hand.

## Enrollment

Fields: id, studentId (foreign key to User), courseId (foreign key to Course), enrolledAt, progressPercent, completed (boolean). This is the single most important modeling decision in the whole product, so it is worth being explicit about why it exists as its own entity. A student enrolling in a course is, on the surface, another many to many relationship, a student can enroll in many courses, a course has many students. The naive way to model that, the way the reference projects modeled their many to many example, is a bare join with no data of its own. But an enrollment is never just a link, it carries its own facts, when it happened, how far along the student is, whether they finished. The moment a relationship needs to carry data about itself rather than just connecting two things, it stops being a simple many to many and becomes its own first class entity with two foreign keys, which is exactly what Enrollment is. This is the concept the eight reference projects never actually showed you, and it is one of the most common real world modeling situations you will run into.

## Order

Fields: id, studentId (foreign key to User), courseId (foreign key to Course), enrollmentId (foreign key to Enrollment, set only once the enrollment is actually created), amount, status (an enum of PENDING, PAID, or FAILED), createdAt. An order is one to one with the enrollment it produces, exactly one successful order creates exactly one enrollment, and the two are created together inside a single database transaction, covered in [09-phase-5-PRD-orders-and-payments.md](09-phase-5-PRD-orders-and-payments.md). This is the same discipline the `Employee` and `Profile` one to one example in the MongoDB reference project touched on conceptually, but here the stakes of getting it wrong are concrete, a transaction failure must never leave a paid for course with no enrollment, or an enrollment that was never actually paid for.

## Review

Fields: id, rating (an integer from 1 to 5), comment, userId (foreign key to User), courseId (foreign key to Course), createdAt. Two separate one to many relationships meet at this one entity, a user has many reviews, and a course has many reviews. A review additionally requires, as a business rule enforced in the service layer rather than the schema, that the reviewing user actually has a completed or in progress Enrollment for that course, which is the first place in the project where a relationship (Enrollment) gates whether a completely different relationship (Review) is even allowed to be created.

## Notification, the one entity in MongoDB

Fields: id, userId (a plain string, not a relational foreign key, since it points across databases into the Postgres User table and Mongoose has no way to enforce that), type (an enum-like string such as NEW_ENROLLMENT or NEW_REVIEW), message, read (boolean, defaulting to false), metadata (a small embedded object, for example the specific courseId and enrollmentId that triggered the notification, using exactly the embedding pattern from `User`/`Address` in the MongoDB reference project), createdAt and updatedAt from `{ timestamps: true }`. Notifications are deliberately not relationally enforced against the Postgres `userId`, because enforcing it would require a cross database transaction, which is exactly the kind of complexity polyglot persistence is supposed to let you avoid for data that does not need that level of guarantee, a lost or slightly stale notification is an acceptable failure mode in a way a lost enrollment or a lost payment never is.

## The full relationship shape summary

One to many: User (instructor) to Course. User to Review. Course to Review. Order to itself is not applicable, but Enrollment to Order is one to one.

Many to many: Course to Category, through a bare join table, the closest LearnBridge gets to the simple pattern the reference projects already showed you. User (student) to Course, through Enrollment, which is many to many with attached data, the pattern the reference projects never showed you.

One to one: Enrollment to Order.

Cross database reference, intentionally unenforced: Notification to User.
