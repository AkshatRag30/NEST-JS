# 06. Shared utilities: retry, the HTTP client, mail, and the event system behind every email

## `retryWithBackoff`, a small helper for a codebase full of flaky third party calls

`src/@core/utils/retry/retry-with-backoff.util.ts` is exactly what its name says, a generic function that wraps any async call and automatically retries it a few times if it fails in a way that looks temporary, waiting longer between each attempt.

```ts
export async function retryWithBackoff<T>(fn: () => Promise<T>, options: RetryOptions = {}): Promise<T> {
    const { maxRetries = 3, baseDelayMs = 500, isRetryable = DEFAULT_IS_RETRYABLE } = options;
    let attempt = 0;
    for (;;) {
        try {
            return await fn();
        } catch (error) {
            if (!isRetryable(error) || attempt >= maxRetries) { throw error; }
            const jitter = Math.random() * baseDelayMs;
            await new Promise((resolve) => setTimeout(resolve, baseDelayMs * 2 ** attempt + jitter));
            attempt++;
        }
    }
}
```

The default `isRetryable` check only treats a 429 (rate limited) response as worth retrying, `error?.code === 429 || error?.status === 'RESOURCE_EXHAUSTED' || error?.response?.status === 429`, on the reasoning that a rate limit is a genuinely temporary problem, waiting and trying again is likely to succeed, whereas a 400 or a 404 means the request itself was wrong and retrying it would just fail the same way again. The delay between attempts doubles each time (`baseDelayMs * 2 ** attempt`), a standard exponential backoff, with a bit of random jitter added on top so that if several requests all failed at the same moment, they do not all retry at exactly the same moment too and immediately overwhelm whatever they are calling a second time.

Given a codebase this note's own README already pointed out relies on twelve separate blockchain and registrar integrations, each one a real network call to someone else's API that can be temporarily rate limited or briefly unavailable, this is exactly the kind of small, generic tool worth having in one shared place rather than every integration module writing its own retry loop slightly differently. A quick check of how often it is actually imported across the codebase shows it is used in only a small handful of places today, which is worth noticing honestly rather than assuming a utility this well built must be everywhere, a good tool existing is not the same as a good tool being adopted consistently.

## `httpClient`, the one shared axios instance meant to replace a raw `axios` import

`src/@core/utils/http-client.util.ts` is short enough to read in full.

```ts
export const DEFAULT_HTTP_TIMEOUT_MS = 8000;
export const httpClient = axios.create({ timeout: DEFAULT_HTTP_TIMEOUT_MS });
```

The comment above it explains exactly why this exists rather than every module just calling `axios.get(...)` directly: "Axios has no default timeout, so a hung third-party endpoint would otherwise block the request indefinitely instead of failing fast." A raw `axios` call with no explicit timeout will wait forever if the server on the other end accepts the connection but never actually responds, which for a backend juggling a dozen external blockchain and registrar APIs is a real risk, one slow provider could tie up a request handler indefinitely. `httpClient` bakes an 8 second timeout into the instance itself, so every call site gets that protection automatically unless it deliberately overrides `timeout` in its own request config, which axios allows on a per call basis. The spec file for this utility proves the override actually works, by starting a real local HTTP server that never responds and confirming a per-call `{ timeout: 100 }` fails in around 100 milliseconds rather than waiting for the 8 second default.

This is used across the codebase far more than `retryWithBackoff`, thirty seven files import it, everything from `PinataService`'s IPFS calls in note 04, to `IpfsUserService`'s Gigapub calls, to `TasksService`'s CoinGecko price lookup in note 05, all route through this one shared instance rather than each one configuring its own axios client from scratch.

## `MailService` and the event driven system that sits behind it

Every "you've got mail" moment anywhere in this application, a welcome email, a password reset, an order confirmation, an affiliate payout notice, ultimately funnels through one method, `MailService.sendMail` (`src/@core/utils/mail/mail.service.ts`).

```ts
async sendMail<T>(to: string, subject: string, title: string, mailType: string, partialContext: T): Promise<MailResponse> {
    handlebars.registerHelper('whichMail', function () { return mailType; });
    await this.mailerService.sendMail({
        to, from: `${emailHeader} <${sender}>`, subject,
        template: __dirname + '/templates-v2/rootmail',
        context: { title, partialContext: partialContext ? { ...partialContext, sender } : undefined, email: to, websiteLink: webClietnMachine, preheaderText: subject, unsubscribe_url: `${webClietnMachine}/profile/user` }
    }).then(() => { result = { success: true, message: 'Send Success' }; }).catch((error) => { result = { success: false, message: error.message }; });
    return result;
}
```

There is exactly one Handlebars email template in this app, `templates-v2/rootmail.hbs`, which is the shared shell every email uses, the Endless Domains logo, the footer with social links and an unsubscribe link, all the boilerplate every marketing or transactional email needs. The single line that makes one shared shell work for dozens of completely different email bodies is this one, near the middle of the template.

```html
{{> (whichMail) partialContext}}
```

This is a Handlebars dynamic partial. `whichMail` is a helper registered fresh on every single `sendMail` call, returning whatever `mailType` string was passed in (a value from the `MailTypes` enum, `welcome`, `forgot-password`, `domain-order-completed`, and so on), and Handlebars resolves that string into the name of a separate partial template file to render inside the shell, with `partialContext` as its data. In practice this means every distinct kind of email in the whole application is its own small partial template file living alongside `rootmail.hbs`, and `MailService` itself never needs to know or care which specific one it is rendering, it just passes through whatever `mailType` and `partialContext` its caller gave it. `MailTypes` (`mail/enum/mail-type.enum.ts`) is the enum of every one of those template names that exists today, and it is a long list, base account lifecycle emails, order lifecycle emails, refund and payment failure emails, affiliate program emails, and newer, app specific ones like waitlist and USDT claim notifications.

`EmailEventService` (`utils/events/email-event.service.ts`) is the one extra layer that sits between a feature module and `MailService` itself, and it is worth understanding exactly what it buys.

```ts
sendMail<T>(mailType: MailTypes, payload: BaseEmailDto<T>): void {
    const { to, subject, title, partialContext } = payload;
    this.mailService.sendMail(to, subject, title, mailType, partialContext)
        .then(() => { logger.log(`${mailType} - email sent`); })
        .catch((err) => { logger.error(err); });
}
```

Notice this method's return type is `void`, not a `Promise`. It calls `mailService.sendMail` and attaches a `.then`/`.catch` purely for logging, but it never awaits that promise itself, meaning the caller gets control back immediately without ever waiting for the actual email to finish sending. This fire and forget shape is deliberate: a user completing a checkout should not have their whole request hang on whether SendGrid's servers happen to respond quickly at that exact moment, sending the confirmation email is important, but it should never be the thing standing between a user and a successful looking response.

`UserEvent` (`utils/events/handlers/user.event.ts`) is where that fire and forget call actually gets triggered, dozens of `@OnEvent(...)` decorated handler methods, one per event name defined in `events.constants.ts`, each one just calling `emailEventService.sendMail` with the matching `MailTypes` value. The event names themselves (`VERIFY_EMAIL`, `WELCOME`, `DOMAIN_ORDER_COMPLETED`, and so on) are what every other feature module across the entire application actually emits, through Nest's own `EventEmitter2`, whenever something worth emailing a user about happens. A feature module finishing an order, for instance, never has to import `MailService` or know anything about Handlebars templates at all, it just emits a `domain.order.completed` event with the right payload DTO, and this file, sitting in a completely different, shared part of the codebase, is what turns that event into an actual email. This is the event driven decoupling this cluster's README promised, and it is the real reason a feature module never has to "know how an email actually gets sent."

`LogSnagEvent` (`handlers/logsnag.event.ts`) rides along on the exact same `EventEmitter2` mechanism but for a different destination, LogSnag, a third party product analytics and alerting tool, rather than an email. It listens for `LOGSNAG_TRACK`, `LOGSNAG_IDENTIFY`, and two insight tracking events, and deliberately disables itself entirely unless the running container's name is exactly `endless-production`, a safeguard against a staging or development environment accidentally polluting the real production analytics dashboard with test events.

The `email-test-trigger` folder (`src/@core/utils/email-test-trigger/`) is a small, superadmin only diagnostic tool built entirely for engineers, not real users, it lets someone manually fire any single email template, every template at once, or a fixed list of previously troublesome ones, at a real inbox, without needing to trigger the real underlying business event first. `NonProductionEmailTestGuard` refuses every route in this controller outright if `ENV` is `production`, since these endpoints exist purely to make iterating on an email template's look and feel faster during development, not something that should ever be reachable on the live site.

## `normalizeEmail`, the small function that keeps two different looking emails pointing at one account

`src/@core/utils/email-normalizer.util.ts` solves a specific, well known Gmail quirk, that Gmail itself treats dots inside the part of an address before the `@` as meaningless, and treats anything after a `+` as an ignorable tag, so `guru.prasad@gmail.com`, `guruprasad@gmail.com`, and `guruprasad+work@gmail.com` all deliver to the exact same real inbox.

```ts
if (domain === 'gmail.com' || domain === 'googlemail.com') {
    localPart = localPart.replace(/\./g, '');
    const plusIndex = localPart.indexOf('+');
    if (plusIndex > -1) { localPart = localPart.substring(0, plusIndex); }
    domain = 'gmail.com';
}
```

Without this normalization, a user could accidentally, or deliberately, register two separate accounts that both actually deliver email to the same person, sidestepping any "one account per email" rule the product wants to enforce, or double claiming something that should only be claimable once per real person. `AiQueryService.resolveUserId`, covered in note 03, is a real, live example of this function being relied on elsewhere in the codebase, an admin looking up a user by a Gmail address with dots or a plus tag in it still resolves to the correct underlying account, with an explicit fallback to a raw lowercase match for any older rows that were written before this normalization rule existed.

## The two `node20-*.spec.ts` files, a Node version upgrade's safety net

`node20-crypto-primitives.spec.ts` and `node20-auth-hash-primitives.spec.ts` are not testing anything specific to this application's own business logic, they are testing that the Node.js runtime itself, on version 20, still correctly provides the low level cryptographic primitives this app depends on everywhere else. The first confirms `crypto.randomUUID()`, `crypto.randomBytes(32)`, and the newer Web Crypto API's `crypto.subtle.digest('SHA-256', ...)` all behave as expected. The second confirms `argon2.hash`/`argon2.verify` and `bcrypt.hash`/`bcrypt.compare`, both native addon backed password hashing libraries, still round trip correctly, a hash created by one call can be verified as correct by the other and correctly rejected against the wrong password.

Both are best understood as regression tests written specifically around a Node.js version upgrade, argon2 and bcrypt both rely on compiled native addons that are sensitive to exactly which Node version and platform they were built against, and a runtime upgrade is one of the more common ways a password hashing library can silently start failing in production. Having an explicit, fast test suite that just answers "do these fundamental primitives still work at all under this Node version" is a genuinely good practice before trusting a version bump anywhere near authentication code, and it is a good example of tests functioning as living documentation of what a low level dependency is actually expected to do, exactly the reason this cluster's brief singled these two files out as worth reading even though they are test files.
