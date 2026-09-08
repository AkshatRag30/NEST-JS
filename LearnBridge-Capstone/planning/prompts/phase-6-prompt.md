# Phase 6 Prompt: Notifications with MongoDB

Use this once phase 5 is complete and you have watched at least one forced payment failure and one real success through the running app.

Copy everything inside the fenced block below into Claude Code as one message.

```
Before writing any code, read these files completely: planning/04-data-model-and-relationships.md (the Notification section) and planning/10-phase-6-PRD-notifications-mongodb.md. This is phase 6, the first and only phase that writes to MongoDB instead of Postgres.

Build a NotificationsModule entirely separate from the Prisma backed modules, injecting only a Mongoose model. Create a Mongoose schema for Notification using @Schema and @Prop decorators from @nestjs/mongoose, with {timestamps: true} enabled, fields for userId (a plain string, this is intentionally not a Postgres foreign key), type (a string, used with values like NEW_ENROLLMENT and NEW_REVIEW), message, read (a boolean defaulting to false), and an embedded metadata object, a separate small class also decorated with @Schema and @Prop but with no schema factory of its own since it only ever exists nested inside a Notification, containing optional fields like courseId, enrollmentId, and reviewId depending on the notification type. Register this schema with MongooseModule.forFeature in the NotificationsModule.

The NotificationsService needs a create method taking a userId, type, message, and metadata object, a findMine method scoped strictly to one userId sorted with the newest first, and a markRead method that updates a notification only if both its id and its userId match the caller, using a single findOneAndUpdate call with both conditions in its filter rather than a separate existence check followed by a separate update, if the query matches nothing, throw a NotFoundException, and do not reveal whether the notification exists for someone else, the response should look identical whether it never existed at all or belongs to another user.

Build a NotificationsController exposing GET /notifications and PATCH /notifications/:id/read, both requiring authentication with no role restriction, always scoped to req.user.id.

Now wire this into the modules that already exist. In OrdersService, after the transaction that creates a PAID order and an Enrollment successfully commits, and only after it commits, call NotificationsService to create a NEW_ENROLLMENT notification for that course's instructor, wrapped in its own try catch block, logging any failure clearly but never allowing that failure to propagate back to the client or affect the already committed order and enrollment in any way, the order and enrollment must succeed regardless of whether the notification write succeeds. Do exactly the same thing in ReviewsService after a review is successfully created, sending a NEW_REVIEW notification to that course's instructor. Inject NotificationsService into both OrdersModule and ReviewsModule properly through their module imports and constructors, this is a real cross module dependency, not a shortcut.

Write a focused unit test for NotificationsService.markRead, using a mocked model, proving that a notification belonging to a different userId than the one attempting to mark it read results in a NotFoundException rather than succeeding. Write a second test on OrdersService specifically proving that when the injected NotificationsService mock is made to throw an error, OrdersService.create still resolves successfully with the PAID order and enrollment intact, this is the single most important test in this phase.

When you are done, tell me exactly which two files you added the try catch wrapped notification calls to, and confirm, from your own testing, that stopping MongoDB entirely while leaving Postgres running still allows a student to successfully complete an order and get enrolled, just without a notification appearing afterward.
```
