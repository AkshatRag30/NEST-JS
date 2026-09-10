# 02. Signature Verification, Compared Across All Four Payment Webhooks

## The problem a webhook endpoint has to solve

A webhook route has to be public. Stripe's servers, Cryptomus's servers, CoinGate's servers, none of them can log into this app as a user, so the route cannot sit behind the normal access token guard the rest of the API uses. But that same openness means anyone on the internet can send a POST to `/api/v1/stripe-webhook/stripe-webhook` or `/api/v1/cryptomusWebhook/` with a handcrafted body claiming a fake order just got paid. If the backend trusted the body's contents at face value, an attacker could mark their own unpaid order as completed and walk away with a domain for free. The entire point of webhook signature verification is closing that gap, proving cryptographically that the request genuinely originated from the provider and was not tampered with in transit, using a secret only the backend and the provider know, before a single line of business logic runs on the payload.

This is also exactly why `main.ts` captures `req.rawBody` during body parsing, covered in note 02 of the architecture notes. Most signature schemes sign the exact raw bytes of the request body. Once Express has parsed those bytes into a JavaScript object, re-serializing that object to compute a signature is not guaranteed to produce the same bytes back, key order, whitespace, and number formatting can all shift. Having the original `Buffer` available separately is what makes a byte-for-byte signature check possible at all.

With that framing, here is what each of the four payment webhook receivers in this codebase actually does, in order from best to worst.

## `ai-subscription-webhook`, the one done properly

`ai-subscription-webhook.controller.ts` grabs the raw body explicitly:

```ts
async handleWebhook(
    @Req() req: Request,
    @Headers('stripe-signature') signature: string,
) {
    const payload = (req as any).rawBody ?? req.body;
    await this.webhookService.handleStripeEvent(payload, signature);
}
```

`ai-subscription-webhook.service.ts` then reimplements Stripe's own signature scheme by hand, `verifyAndParseStripeSignature`:

```ts
const payload = Buffer.isBuffer(rawBody) ? rawBody.toString('utf8') : rawBody;
const signedPayload = `${timestamp}.${payload}`;
const expected = crypto.createHmac('sha256', secret).update(signedPayload, 'utf8').digest('hex');

const expectedBuf = Buffer.from(expected, 'hex');
const matched = v1Signatures.some((sig) => {
    const sigBuf = Buffer.from(sig, 'hex');
    return sigBuf.length === expectedBuf.length && crypto.timingSafeEqual(new Uint8Array(expectedBuf), new Uint8Array(sigBuf));
});
if (!matched) throw new Error('Stripe webhook signature mismatch');
```

Walking through why each piece matters, the `stripe-signature` header is a comma separated string like `t=1699999999,v1=abc123...`, so the code first splits out the timestamp and one or more `v1` values. It rejects the request outright if that timestamp is more than five minutes old (`STRIPE_SIGNATURE_TOLERANCE_SECONDS = 300`), which defends specifically against someone capturing a valid, old webhook call and replaying it later. It then recomputes the expected signature by HMAC-SHA256 hashing the literal string `timestamp.rawBody` with a secret pulled from configuration, `STRIPE_AI_WEBHOOK_SECRET`, never a value hardcoded in the source. Finally, and this is the detail that is easy to skip past, it compares the computed and received signatures with `crypto.timingSafeEqual` rather than `===`. A plain string comparison returns as soon as it finds the first mismatched character, which means how long the comparison takes leaks how many leading characters were correct, an attacker who can measure response timing precisely enough can use that leak to guess a valid signature one byte at a time. `timingSafeEqual` always takes the same amount of time regardless of where a mismatch occurs, closing that side channel. This is, character for character, the textbook correct way to verify a webhook signature, and it is worth remembering this pattern well, because it is exactly what you would reach for on any backend job that involves receiving webhooks.

## `cryptomus-webhook`, adequate but with real rough edges

`cryptomus-webhook.service.ts` verifies every incoming payload before doing anything else with it:

```ts
private verifySignatue(data: Record<string, any>): boolean {
    const remote: string = data['sign'];
    delete data['sign'];
    return remote === this.cryptomusIntegrationServiceInterface.getSignature(data);
}

async cryptomusWebhook(reqBody: ...): Promise<void> {
    const body = reqBody;
    if (!this.verifySignatue(body)) {
        logger.error('Signature verification failed');
        throw new Error('Signature verification failed');
    }
    ...
}
```

`getSignature`, in `cryptomus-integration.service.ts`, computes the expected value like this:

```ts
getSignature(data: Record<string, string | number | boolean>): string {
    return createHash('md5')
        .update(Buffer.from(JSON.stringify(data)).toString('base64') + this.CRYPTOMUS_TOKEN)
        .digest('hex');
}
```

So Cryptomus sends its own computed `sign` field inside the JSON body itself, and the backend removes that field, base64 encodes the remaining JSON, appends a shared secret token (`CRYPTOMUS_TOKEN`, pulled from configuration rather than hardcoded), and MD5 hashes the result, then checks whether that matches what Cryptomus sent. If it does not match, the webhook throws before anything is saved.

This does genuinely stop an attacker who does not know the shared token from forging a payload, so it is real verification, not the absence of it. But it has two weaknesses worth naming plainly. First, it hashes the already-parsed `data` object rather than the original raw request bytes, using `JSON.stringify(data)` on the deserialized body instead of `req.rawBody`. This only works reliably if Cryptomus's own signing step serializes the fields in the exact same order Node's JSON parser happens to preserve them in, which is fragile in a way a raw-byte comparison is not. Second, the final comparison is a plain `===` on two strings, not a constant time comparison, so it carries the same timing side channel that `ai-subscription-webhook` specifically avoided with `timingSafeEqual`.

## `stripe-webhook`, no verification at all

This is the important negative finding. `stripe-webhook.controller.ts`, the receiver for ordinary domain purchase payments, the highest value webhook in the entire app, does this:

```ts
@Post('/stripe-webhook')
@HttpCode(HttpStatus.OK)
public async getDataFromUDWebhook(@Body() reqBody: any): Promise<Response> {
    ...
    await this.stripeWebhookServiceInterface.stripeWebhook(reqBody);
    return new Response('success');
}
```

There is no `@Headers('stripe-signature')` parameter here at all. `stripe-webhook.service.ts`'s `stripeWebhook` method goes straight from `if (data?.data?.object?.id)` into looking up an order by that id and processing whatever `data.type` says happened, `payment_intent.succeeded`, `payment_intent.payment_failed`, and so on. Nowhere in this file is Stripe's signing secret referenced, nowhere is `req.rawBody` read, nowhere is an HMAC computed. Functionally, anyone who can guess or has previously seen a real Stripe `payment_intent` id for an order (which is not secret information, a customer's own browser sees it during checkout) can POST a fabricated `payment_intent.succeeded` event straight to this endpoint and the backend will process it as if Stripe itself sent it, transferring the domain to whatever wallet address happens to be on that order.

This matters more than it might first appear, because `ai-subscription-webhook` proves the team clearly knows how to verify a Stripe signature correctly, that code exists in the same codebase, in the same webhooks folder. The gap here reads as the older, original payment webhook simply predating that pattern being adopted, rather than a deliberate choice. Anyone picking up this codebase should treat closing this gap, adding `stripe.webhooks.constructEvent` (Stripe's own SDK helper, which does exactly what `ai-subscription-webhook` reimplemented by hand) against `req.rawBody`, as a priority, not a nice to have.

## `udwebhook`, verification computed and then thrown away

This is the second, and in some ways stranger, finding. `udwebhook.service.ts`'s `udV3Webhook` method opens like this:

```ts
async udV3Webhook(headers: Record<string, string>, reqBody: any): Promise<boolean> {
    const signature = headers['x-ud-signature'];
    const rawBodyBytess = Buffer.from(JSON.stringify(reqBody), 'utf-8');
    const computedSignature = createHmac('sha256', '<hardcoded secret literal>').update(rawBodyBytess).digest('base64');

    let orderId;

    // if (signature == computedSignature) {
    const body = reqBody;
    ...
```

(The literal secret string has been replaced above, it is a real value sitting directly in the source file in the actual codebase, not something to repeat here or anywhere else.)

Two separate problems sit in these five lines. The first is that the actual check, `if (signature == computedSignature)`, is commented out. The code goes to the trouble of reading the `x-ud-signature` header and computing what the signature should be, and then simply never compares the two, execution falls straight through to `const body = reqBody` and processes the payload unconditionally regardless of whether `signature` matches, is missing, or is garbage. This makes the entire signature computation dead code as far as security goes, it looks like verification is happening if you skim the file, but nothing is actually being enforced.

The second problem is that the secret used to compute that (currently unchecked) signature is a literal string written directly into the TypeScript source, not read from `ConfigService` or AWS Secrets Manager the way every other credential in this app is, per the configuration note. That means the secret is sitting in plain text in version control history, visible to anyone with repository access, present tense, right now, independent of whether the comparison above is ever turned back on. Even after uncommenting that `if`, this secret would need to be rotated and moved into Secrets Manager before the fix means anything.

## The pattern across all four, in one place

| Receiver | Verifies a signature? | Uses `req.rawBody`? | Secret source | Notes |
|---|---|---|---|---|
| `ai-subscription-webhook` | Yes, HMAC-SHA256, timing safe compare, timestamp tolerance | Yes | `ConfigService` (`STRIPE_AI_WEBHOOK_SECRET`) | The correct reference implementation |
| `cryptomus-webhook` | Yes, MD5 hash compare | No, hashes the parsed body | `ConfigService` (`CRYPTOMUS_TOKEN`) | Real, but a plain `===` compare and reliant on stable JSON key order |
| `coingate-webhook` | No signature check found in this codebase | No | n/a | Trusts the payload's own `token` field to look up an invoice, with no cryptographic proof it came from CoinGate |
| `stripe-webhook` | No | No | n/a | The highest value target of the four, and the least protected |
| `udwebhook` | Computed but never compared, commented out | No, hashes the parsed body | Hardcoded literal in source | Looks verified at a glance, is not, and leaks a secret regardless |

If you take one thing from this file, it is that "this codebase has webhook signature verification" is not a single fact you can answer yes or no to here, it depends entirely on which of the five receivers you are looking at, and three of the five have a real problem worth flagging to whoever owns this code.
