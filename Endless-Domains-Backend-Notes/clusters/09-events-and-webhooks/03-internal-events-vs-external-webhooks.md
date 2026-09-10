# 03. Internal Events Versus External Webhooks, and a Naming Collision That Makes It Worse

## The distinction itself

A webhook is a message this backend receives, sent by a completely separate system it does not control, arriving over the public internet as an HTTP request, carrying no guarantee it is genuine until proven with a signature check (see note 02). Everything in `src/components/webhooks` is this.

An internal event is a message this backend sends to itself. One piece of code, already running inside this same Node process, midway through handling some request, calls `this.eventEmitter.emit('something.happened', payload)`. Some other piece of code, also inside this same process, registered ahead of time with `@OnEvent('something.happened')`, wakes up and reacts. No network call happens at all, no signature is needed, because nothing external was ever involved, it is one part of this app talking to another part of this app, decoupled only so that the part raising the event does not need to know or care who, if anyone, is listening.

The mechanism underneath the internal version is `@nestjs/event-emitter`, turned on once, application wide, in `app.module.ts`:

```ts
EventEmitterModule.forRoot(),
```

Once that is registered, any injectable class in the app can inject `EventEmitter2` and call `.emit(...)` on it, and any injectable class, anywhere else in the app, can decorate a method with `@OnEvent(...)` and have Nest wire it up as a listener automatically, no explicit registration needed beyond the decorator itself.

## Tracing one all the way through

The clearest way to see this is to follow one event from where it is raised to where it lands, across files that have no import relationship to each other at all beyond the shared event name.

It starts in `auth.service.ts`, right after a user logs in successfully:

```ts
this.eventEmitter.emit(`userlogin.UpdateDomainDetailTable`, new UpdateDomainDetailTableAfterLoginEvent(user.id));
```

`UpdateDomainDetailTableAfterLoginEvent`, the payload class being passed here, lives in `src/components/event/update-domain-detail-table-after-login.event.ts`, and it is about as small as a class can be:

```ts
export class UpdateDomainDetailTableAfterLoginEvent {
    constructor(public readonly userId: string) {}
}
```

That is genuinely all `src/components/event` holds, a handful of tiny classes like this one, each just a typed payload shape for a specific internal event, `domain-sync-completed.event.ts` and `start-claim-domain-background-process.event.ts` sit alongside it the same way. They exist purely so that whoever writes the `@OnEvent` handler on the other end gets a real, typed object instead of an untyped blob, there is no logic in this folder at all, just shapes.

The listener side is a method on `fetch-domain.service.ts`:

```ts
@OnEvent('web3UserLogin.UpdateDomainDetailTable', { async: true })
async updateDomainDetialTable(payload: UpdateDomainDetailTableAfterLoginEvent): Promise<void> {
    ...
    this.eventEmitter.emit('domain.sync.completed', new DomainSyncCompletedEvent(payload.userId, data));
}
```

(`auth.service.ts` also has its own listener registered directly on the same class for the plain email login variant, `@OnEvent('userlogin.UpdateDomainDetailTable', { async: true })`, and `web3-auth.service.ts` and `google-auth.service.ts` each emit and listen to their own login method's version of the same event. Three separate login flows, three separate event names, all converging on the same idea, refresh a user's domain detail table after they log in, decoupled from the login method itself doing that work inline.)

And it does not stop there. That handler itself raises a second, different event once it finishes, `domain.sync.completed`, which `domain-tenure.service.ts` picks up:

```ts
@OnEvent('domain.sync.completed', { async: true })
async handleDomainSyncCompleted(payload: DomainSyncCompletedEvent): Promise<void> { ... }
```

So one user action, logging in, sets off a small chain of two internal events, each handler doing its own piece of work and then, if needed, raising the next event in the chain, without any of the three services (`auth.service.ts`, `fetch-domain.service.ts`, `domain-tenure.service.ts`) needing to import each other directly or know the others exist. That decoupling is the entire reason this pattern exists.

## Where webhooks and internal events meet

The webhook services in this cluster lean on this same mechanism constantly, and it is worth being precise about exactly where the boundary sits. When `stripe-webhook.service.ts` finishes registering a domain successfully, it does this:

```ts
this.eventEmitter.emit(
    DOMAIN_ORDER_COMPLETED,
    new DomainOrderCompletedDto({ to: user.email, subject: 'Domain Order Completed', ... })
);
```

`DOMAIN_ORDER_COMPLETED` is just a string constant defined in `src/@core/utils/events/constants/events.constants.ts`. The listener for it lives in `src/@core/utils/events/handlers/user.event.ts`, outside this cluster's folders but worth knowing about since so many of this cluster's emits target it:

```ts
@OnEvent(DOMAIN_ORDER_COMPLETED, { async: true })
handleDomainOrderCompleted(payload: DomainOrderCompletedDto): void {
    this.emailEventService.sendMail(MailTypes.DOMAIN_ORDER_COMPLETED, payload);
}
```

So the external webhook (Stripe telling this app a payment succeeded) is the trigger, but everything that happens after `this.eventEmitter.emit(DOMAIN_ORDER_COMPLETED, ...)` runs is purely internal. Stripe never knows or cares that an email gets sent, that a `LOGSNAG_TRACK` analytics event fires, or that any of this happens at all, its job ended the moment its HTTP request to the webhook controller returned a 200. Everything downstream of that `emit` call is this app talking to itself. The same `user.event.ts` file listens for dozens of these constants this way, `STRIPE_PAYMENT_FAILURE`, `STRIPE_REFUND_FAIL`, `SUBSCRIBER`, `GENERIC_ORDER_PROCESSING_ERROR`, and many more, each one a one line `@OnEvent` handler that hands the payload to `EmailEventService.sendMail`. A second handler file, `logsnag.event.ts`, listens for a smaller set, `LOGSNAG_TRACK`, `LOGSNAG_IDENTIFY`, `LOGSNAG_INSIGHT_INCREMENT`, and forwards those to a third party analytics service called LogSnag instead of to email. Both are internal listeners in exactly the same sense, they simply act on different families of internal events.

A useful rule of thumb: if you see `this.eventEmitter.emit(...)` or `@OnEvent(...)` in a file, you are looking at this app talking to itself, no network request happened, nothing needs a signature. If you see a `@Controller` under `src/components/webhooks` receiving a `@Post`, you are looking at an external system talking to this app, and note 02's question, is this verified, becomes relevant.

## The naming collision worth calling out directly

Here is the part that genuinely confuses a first read. `src/components/events` (plural, no relation to `event` or `event-subscribe`) is not this event bus at all. Open `event.entity.ts` and you find this:

```ts
@Entity({ name: 'tbl_events' })
export class EventEntity extends BaseEntity {
    title: string;
    description: string;
    type: string;       // virtual | regional | global
    location: string;
    date: Date;
    status: string;     // upcoming | ongoing | completed
    featured: boolean;
    imageUrl: string;
    isHighlight: boolean;
    isCommunity: boolean;
}
```

This is a marketing CMS table for real world events the company runs or takes part in, conferences, community meetups, highlight reels with a photo and video gallery. `event.controller.ts` is a fairly ordinary admin guarded CRUD controller, `POST /events`, `PUT /events/:id`, `DELETE /events/:id`, plus a couple of public, unauthenticated read routes for the marketing site to render an events page from, `GET /events/get-highlights-events`, `GET /events/branding/public`. None of this has anything to do with `EventEmitter2`, `@OnEvent`, or anything raised and reacted to inside the backend. It is simply that "event" is an overloaded word in English, and this company happens to run both kinds, software events dispatched through an event bus, and marketing events people can register interest in attending. `event-subscribe`, covered in note 05, is the signup list for that second, human meaning of the word, people who want to be told when the company announces a new conference or community meetup, which again has no relationship to `EventEmitter2` at all.

If you remember nothing else from this file, remember that "event" in this codebase can mean either of two unrelated things depending on which folder you are standing in, and the folder name plural versus singular (`events` the marketing CMS, `event` the tiny internal payload shapes folder) is, unhelpfully, almost the only clue distinguishing them at a glance.
