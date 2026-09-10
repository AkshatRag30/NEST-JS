# 05. The Marketing Events CMS, Newsletter Subscriptions, and the Mail Audit Table

## `events`, `event-subscribe`, and `event.controller.ts`, a small content management system

As note 03 explains, `src/components/events` (plural) has nothing to do with `EventEmitter2`. It is a straightforward, admin managed content system for real world company events, conferences, community meetups, highlight reels. `EventEntity` carries a title, description, a type of `virtual`, `regional`, or `global`, a location, a date, a status of `upcoming`, `ongoing`, or `completed`, a `featured` flag, and two more specific booleans, `isHighlight` and `isCommunity`, which is how `event.service.ts` tells apart the "create a highlights style event" and "create a community event" endpoints from a plain event, `createHighlightsEvent` and `createCommunityEvent` are both thin wrappers around the same `create` logic that just set those two flags differently going in.

`event.controller.ts` mixes admin only routes with public ones on purpose, and the split is worth noticing because it is a clean example of the pattern. Creating, updating, deleting an event, and uploading its images, all sit behind `@UseGuards(SuperAdminAccessGuard)`. But `GET /events/get-highlights-events`, `GET /events/get-community-events`, `GET /events/branding/public`, and `GET /events/:id/details/public` carry no guard at all, because these are the routes the public marketing site itself calls to render an events page for anyone visiting it, no login involved. `event-branding.service.ts` and `event-detail.service.ts` (not shown here in full) round this out, branding covers logos and a small rotating gallery of recent images shared across all events rather than tied to one specific event, and detail covers the richer hero image, about text, and photo or video gallery for one specific event's own page.

`event-subscribe` is the signup list attached to this same marketing feature, people who want to be told when the company announces a new conference or community event. `EventSubscribeEntity` is about as small as an entity gets:

```ts
@Entity({ name: 'tbl_event_subscribe' })
export class EventSubscribeEntity extends BaseEntity {
    @Column({ nullable: false, unique: true })
    email: string;
}
```

The public signup route is worth a close look because of a guard most of the other public facing routes in this cluster do not carry:

```ts
@Post()
@UseGuards(IpRateLimitGuard)
public async subscribe(@Body() dto: EventSubscribeDto): Promise<Response> {
    return new Response(['Subscribed Successfully'], await this.service.createNewEventSubscriber(dto)).setStatusCode(200);
}
```

`IpRateLimitGuard` throttles this specific route per calling IP address, on top of whatever the application wide throttler in `app.module.ts` already does. A public, no login required, email collecting form like this is an obvious target for someone scripting thousands of fake signups, either to spam the underlying mailing list provider or just to pollute the data, so an extra layer of rate limiting here is a deliberate, sensible choice. The rest of the controller is admin only, `GET /event-subscribe` to list subscribers with filtering, `GET /event-subscribe/export-subscribers/csv-list` to download the whole list as a CSV for use in an outside email tool, and `GET /event-subscribe/:id` to look up one subscriber's record.

## `subscribe`, the general newsletter, and the gap between the two

`src/components/subscribe` is the same idea again, one step more general, a plain newsletter signup not tied to any specific conference or event. `SubscribeEntity` is identical in shape to `EventSubscribeEntity`, a unique email column and nothing else. The controller is close to a copy of `event-subscribe`'s, list, export to CSV, fetch one by id, and a public `POST /subscribe` for anyone to sign up.

The gap worth noticing directly: `SubscribeController.subscribe` carries no `IpRateLimitGuard`, or any other guard, at all.

```ts
@Post()
public async subscribe(@Body() dto: SubscribeDto): Promise<Response> {
    return new Response(['Subscribed Successfully'], await this.service.createNewSubscriber(dto)).setStatusCode(200);
}
```

Two public, unauthenticated, email collecting POST routes exist side by side in this codebase, doing almost the same job, and only one of them has rate limiting applied. That is exactly the kind of small, easy to miss inconsistency that is worth flagging if you ever find yourself doing a security pass over a codebase like this one, not because either route is dramatically broken, but because it shows a protection that was clearly considered important enough to add in one place quietly did not get carried over to its near identical sibling.

`subscribe.service.ts` also does something `event-subscribe` does not, it raises an internal event once a new subscriber is saved, tying this module back into note 03's event bus:

```ts
this.eventEmitter.emit(
    SUBSCRIBER,
    new SubscriberDto({
        to: subscriber.email,
        subject: 'Thank You for Subscribing to Our Newsletter!',
        ...
    })
);
this.eventEmitter.emit(LOGSNAG_TRACK, { event: 'Subscriber', channel: 'subscribers', ... } as TrackOptions);
```

`SUBSCRIBER` is picked up by the same `user.event.ts` handler file described in note 03, which sends the actual thank you email, and `LOGSNAG_TRACK` is picked up by `logsnag.event.ts`, which reports the new subscriber to the company's LogSnag analytics dashboard. Neither of those two downstream effects has anything to do with an external request, both are entirely internal, triggered off the back of one successful newsletter signup.

## `mail-send-meta-data`, an audit table with no writer

`MailSendMetaDataEntity` is shaped exactly like what its name promises, a record of one email that was sent, or was meant to be sent:

```ts
@Entity({ name: 'tbl_mail_send_meta_data' })
export class MailSendMetaDataEntity extends BaseEntity {
    mail_type: string;
    user_id: string;
    user_name: string;
    user_email: string;
    data: JSON;      // jsonb
    status: string;
}
```

The intent behind a table like this is easy to guess even without more context, keep a durable log of which mail type went to which user with what data, so a support agent or an ops dashboard can answer "did this customer actually get their order completed email" without needing to dig through CloudWatch logs, and potentially so a scheduled reminder job can check this table first and avoid sending the same reminder to the same person twice.

But the component itself, `mail-send-meta-data.service.ts`, only exposes one method, and it is a read:

```ts
async readAllMailSendMetaDataByStatusAndMailType(mailType: string, status: string): Promise<MailSendMetaDataEntity[]> {
    return await this.mailSendMetaDataRepo.findAllByStatusAndMailType(mailType, status);
}
```

`mail-send-meta-data.controller.ts` exposes exactly that, one admin guarded `GET /mail-send-meta-data` route, filterable by `mailType` and `status`, clearly meant as an internal ops or support tool for browsing this table. Searching the entire `src` tree for `MailSendMetaDataEntity` and for the literal table name `mail_send_meta_data` turns up no other reference anywhere in the current codebase, no service constructs one of these rows, no repository method inserts into it, `EmailEventService`, the shared service every `@OnEvent` mail handler in note 03 actually calls to send mail, does not touch this table either.

Taken at face value, that means this table currently has a read side and an admin endpoint built for it, but no writer anywhere in this codebase. Either something upstream of this repository, a database seed, a retired job, a different service that once wrote here and has since been removed, was responsible for populating it and no longer exists, or this is a table and endpoint built ahead of a write path that has not been wired up yet. Either way, if you go looking at this endpoint expecting a live audit trail of every email this app sends, what you will actually find, as of this codebase, is an empty or stale table with no code path feeding it. That is worth knowing before you rely on it for anything.
