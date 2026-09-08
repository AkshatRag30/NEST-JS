# Phase 6 PRD: Notifications with MongoDB

## Goal

Build the one part of the product that genuinely belongs in a different kind of database than everything else, and use it to revisit every core Mongoose concept from `MongoDB-with-Nest-JS-main`, this time inside a system that already has real relational data sitting next to it.

## Concepts practiced

`@Schema`, `@Prop`, `SchemaFactory.createForClass`, `MongooseModule.forFeature`, `@InjectModel`, embedding a subdocument, and the `{ timestamps: true }` option, all applied to a genuinely new schema rather than copied from the reference project.

## Scope

A `NotificationsModule`, entirely separate from every Postgres backed module, injecting only a Mongoose `Model<NotificationDocument>`. The `Notification` schema has userId, type, message, a read boolean defaulting to false, an embedded metadata subdocument (for example `{ courseId, enrollmentId }`, whichever fields are relevant to that notification's type), and timestamps enabled for createdAt and updatedAt.

Notifications are never created directly by a client request, they are created internally, as a side effect of something else happening elsewhere in the system. When phase 5's order flow successfully creates an enrollment, it also calls a method on `NotificationsService` to create a notification for the instructor of that course, letting them know a new student enrolled. When phase 4's review flow successfully creates a review, it similarly notifies the course's instructor of the new review. These calls cross module boundaries, `OrdersModule` and `ReviewsModule` each need `NotificationsService` injected, which is a real, deliberate practice of dependency injection across feature modules, not just within one.

`GET /notifications` returns the authenticated user's own notifications, newest first, and `PATCH /notifications/:id/read` marks one as read, both scoped strictly to `req.user.id` so one user can never read or modify another user's notifications, which has to be checked by comparing the notification's stored `userId` against the caller, since MongoDB has no foreign key to enforce it for you the way Postgres would.

## API surface

`GET /notifications`, `PATCH /notifications/:id/read`, both authenticated, no role restriction, any logged in user only ever sees their own.

## Acceptance criteria

1. Successfully enrolling in a course (through the phase 5 order flow) results in a new, unread notification appearing for that course's instructor within the same request, verifiable by immediately calling `GET /notifications` as that instructor.
2. A student calling `GET /notifications` never sees another student's or another instructor's notifications, verified by creating notifications for two different users and confirming each only sees their own.
3. Marking a notification as read is permanent and idempotent, calling the endpoint twice on the same notification does not error the second time.
4. If MongoDB is unreachable, verified by intentionally stopping it during local testing, creating an enrollment still succeeds, the notification failure should be caught and logged, not allowed to roll back or block the actual enrollment, since a lost notification is an acceptable failure but a lost enrollment is not, this is the direct, concrete payoff of having kept these two pieces of data in separate databases.

## Explicit trap to avoid

Do not let a `NotificationsService` failure be allowed to fail the transaction that triggered it. Wrap the notification creation call in its own try catch at the call site inside `OrdersModule` and `ReviewsModule`, log the failure, and let the primary operation succeed regardless. If a notification failure were allowed to roll back an enrollment, there would have been no point keeping the two databases separate in the first place.
