# 01. AI Domain Discovery: turning a business idea into a domain name

## Where this lives and why it is its own controller

`src/components/ai/ai-domain-discovery.module.ts`, `ai-domain-discovery.controller.ts`, and `ai-domain-discovery.service.ts` make up a small, self contained feature that is registered in `app.module.ts` as its own `AiDomainDiscoveryModule`, separate from the much bigger `AiModule` that holds the admin assistant and the Domain Advisor. The controller's own comment explains why plainly.

```ts
/**
 * Unauthenticated on purpose — visitors describing a business idea haven't
 * signed up yet, so there's no user to gate on. Kept as its own controller
 * (rather than folded into AiController) because AiController carries a
 * class-level AccessTokenGuard that every route in it inherits.
 */
@Controller('ai/domain-discovery')
export class AiDomainDiscoveryController {
```

`AiController`, covered in note 03, has `@UseGuards(AccessTokenGuard)` at the class level, meaning every route inside it requires a logged in user. This feature is deliberately for people who have not created an account yet, someone typing "I sell handmade leather shoes online" into a search box on the marketing site before they have any reason to sign up, so it needed its own controller with no login requirement at all.

## The two endpoints, and what "direct search" actually spends

There are two routes, `POST /ai/domain-discovery/suggestions` and `POST /ai/domain-discovery/search`, both taking a `businessIdea` string (5 to 500 characters, validated by `AiDomainDiscoveryRequestDto`) and a visitor's IP address pulled off the request.

`suggestOnly` asks Claude for five candidate domain labels and returns them, full stop, no availability checking against any real domain provider. `directSearch` does the same generation step, but then takes only the single strongest candidate (Claude is instructed to rank its five answers strongest first) and actually runs it through `DomainSearchService.getDomainSuggestion()`, the same multi provider fan out across all twelve blockchain and registrar integrations that a real logged in user's domain search uses. The service's own comment is explicit about why it is only ever one candidate, not five.

```ts
// Claude is instructed to rank candidates strongest-first, so the
// first one is the pick that actually gets checked against every
// provider. Only ever one provider fan-out per request, not five —
// that's the whole point, five would multiply external API calls
// (UD, Freename, RPC nodes, ...) fivefold for no real benefit.
```

Checking five candidates against twelve providers each would be sixty external calls for one visitor's one idea. Checking one candidate is twelve. The remaining four candidates come back in the response's `otherCandidates` field, free to look at, costing nothing unless the visitor deliberately searches one of them separately.

## The actual Claude call, and why it forces a tool call rather than free text

`generateCandidates` is the method that talks to Anthropic. It builds a long, carefully worded system prompt (worth reading in full in `ai-domain-discovery.service.ts` if you want to see real prompt engineering for a narrow, specific task) that tells Claude to extract the two or three most literal, concrete keywords from the business idea, keep every candidate at six characters or fewer because "Endless Domains prices shorter labels higher and treats them as premium inventory," and always answer by calling a tool rather than replying in plain text.

```ts
tools: [
    {
        name: CANDIDATE_TOOL_NAME,
        description: `Provide exactly ${CANDIDATE_COUNT} candidate domain labels for the business idea.`,
        input_schema: {
            type: 'object',
            properties: {
                candidates: { type: 'array', items: { type: 'string' }, minItems: CANDIDATE_COUNT, maxItems: CANDIDATE_COUNT }
            },
            required: ['candidates']
        }
    }
],
tool_choice: { type: 'tool', name: CANDIDATE_TOOL_NAME }
```

This is Anthropic's tool use feature being used purely as a structured output mechanism, not to let Claude take an action. `tool_choice: { type: 'tool', name: ... }` forces the model to always respond by calling `provide_domain_candidates` with a JSON array of five strings, rather than sometimes replying in a paragraph of prose that the code would then have to parse. It is a reliable, simple way to get a predictable shape back from a language model, and it shows up again, in a more elaborate form, in the Domain Advisor and the admin tools covered in the next two notes.

Even so, the code does not fully trust the model to follow its own instructions. `cleanCandidates` strips anything that is not a plain lowercase letter, drops anything shorter than two characters or longer than the six character cap, deduplicates, and only then hands the result back. The comment on `MAX_LABEL_LENGTH` explains the reasoning for enforcing this in code rather than truncating a too long answer.

```ts
// Shorter domain labels price higher in Endless Domains' own pricing model,
// so candidates longer than this are dropped even if Claude ignores the
// prompt's length rule, rather than truncated (truncating a real word mid
// way through produces a broken, unbrandable fragment, not a short one).
```

Truncating "expensive" to "expen" would produce a word nobody would ever actually want as a domain. Dropping it entirely and keeping only the candidates that were already short enough on their own is the safer failure mode.

The model used is `claude-haiku-4-5`, not the Sonnet model used everywhere else in the AI module, with a comment explaining the choice directly: "this call only ever produces five short brandable words, and that's exactly the 'simple, speed-critical' case where the fastest tier is the right one." Both the model name and the provider search timeout are read from `ConfigService` with hardcoded fallbacks, so either can be tuned in the AWS secret bundle without a redeploy.

## Two independent guards stacked on both endpoints, and why there are two

Both routes carry `@UseGuards(AiDomainDiscoveryCircuitBreakerGuard, AiDomainDiscoveryRateLimitGuard)`. These solve two different problems and neither one alone would be enough.

`AiDomainDiscoveryRateLimitGuard` (in `src/@core/common/guards/`) stops one visitor from hammering the endpoint. Its own comment explains a real security lesson worth internalizing: an IP address alone is a weak identity, "a visitor could reset just by switching Wi-Fi to mobile data, using a VPN, or bouncing through a proxy." The guard tracks two independent counters, the request IP and a signed, HttpOnly cookie it issues on first visit, and only allows a request through if both counters still have room, five requests per rolling 24 hours each. Switching networks alone does not reset the cookie counter; clearing cookies alone does not reset the IP counter. Both counters live in Postgres, in a `tbl_ai_domain_discovery_rate_limit` table, specifically because this backend runs as multiple ECS tasks behind a load balancer, so an in memory counter on one task would not see traffic landing on a different task. The actual check and increment happens as a single `INSERT ... ON CONFLICT DO UPDATE ... WHERE` statement, which avoids a race condition where two nearly simultaneous requests both read "under the limit" before either one writes its increment.

`AiDomainDiscoveryCircuitBreakerGuard` (also in `src/@core/common/guards/`) protects the feature as a whole rather than any one visitor, because "many different legitimate visitors arriving at once, or a scraper spread across many rotating identities that each individually stay under the per-visitor limit" could still add up to a real cost problem. It reads straight from the `tbl_ai_domain_discovery_log` table, which every call already writes to (whether it succeeded or failed), and trips if the platform sees more than 200 requests in the last hour, more than 2,000 in the last 24 hours, or if the estimated Claude API spend from logged input and output token counts crosses a hardcoded 20 dollar daily budget. The per token cost figures are hardcoded with a comment reminding whoever changes the model later to update them too.

## What gets logged, and why the response looks different outside production

`AiDomainDiscoveryLogEntity` (`tbl_ai_domain_discovery_log`) records the visitor's IP, the business idea text, which of the two approaches was used, the cleaned candidate list, a compact result summary, success or failure, an error message if any, latency, and token counts. This one table is what both guards above actually read to make their decisions, so it is worth remembering it is not just an audit log, it is load bearing infrastructure for the rate limiting and cost control logic.

The response shape for `directSearch` also changes based on environment.

```ts
if (this.isProduction) {
    return { businessIdea, results: results.map(({ candidate, suggestions }) => ({ candidate, suggestions })), otherCandidates, generatedAt: new Date().toISOString() };
}

return { businessIdea, results, otherCandidates, generatedAt: new Date().toISOString(), timingDebugMs: { generation: generationMs, search: searchMs } };
```

Outside production, the response also includes a `timingDebugMs` breakdown of how long the Claude call versus the provider search took, and each per candidate result can carry a `debugNote` explaining exactly why it came back empty, timed out, threw an error, or genuinely resolved with zero results. Both fields are explicitly commented as temporary diagnostic aids for tracking down a real, currently unresolved bug where candidates keep coming back empty, and both are stripped out in production specifically because `debugNote` can leak a raw provider error message to an unauthenticated caller, which would be an information disclosure problem if it shipped.
