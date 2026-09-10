# 03. The admin side of the AI module: tools, insights, reports, and alerts

## The tool registry and dispatcher, the architecture both AI features share

Both the Domain Advisor from note 02 and the admin assistant described in this note are built on the same small, generic plumbing, defined in `src/components/ai/tools/ai-tool.interface.ts`, `ai-tool-registry.service.ts`, and `ai-tool-dispatcher.service.ts`. This is worth understanding on its own before looking at any individual tool, because it explains why adding a brand new AI "personality" to either the admin panel or the user facing advisor is a small, one file change rather than a rewrite.

Every concrete tool, whether it is admin scoped (`UserOrderQueryTool`, `OrderQueryTool`, `CustomerInsightsTool`, `AdminGeneralAssistantTool`) or user scoped (`PortfolioInsightsTool`, `TldKnowledgeTool`, and so on), implements a tiny `IAiTool` interface, a bit of metadata (a unique key, which scope it belongs to, a display name, whether it is enabled, whether it is the default) plus an `execute` method and an optional `stream` method.

```ts
export interface IAiTool {
    readonly metadata: AiToolMetadata;
    execute(ctx: AiToolContext): Promise<AiToolResult>;
    stream?(ctx: AiToolContext): Observable<MessageEvent>;
}
```

`AiModule` collects every concrete tool class into one array and exposes it under a shared injection token.

```ts
const AI_TOOLS_FACTORY_PROVIDER: Provider = {
    provide: AI_TOOLS,
    useFactory: (...tools: any[]) => tools,
    inject: AI_TOOL_CLASSES,
};
```

The module's own comment explains why this factory provider exists rather than something simpler: "NestJS does NOT support Angular-style `multi: true` providers," so there is no built in way to say "give me every provider that implements this interface." Building one factory function that takes every concrete tool as a constructor argument and returns them as a plain array is the workaround, and it means `AiToolRegistryService` can then just loop over that array once at startup and index every tool by its `metadata.key`.

```ts
constructor(@Inject(AI_TOOLS) tools: IAiTool[]) {
    for (const tool of tools) {
        ...
        this.toolsByKey.set(tool.metadata.key, tool);
    }
}
```

`AiToolDispatcherService` is the thin layer `AiController` actually talks to. It validates the incoming question's length, asks the registry for the right tool by key and scope, and calls either `execute` (for the ordinary REST endpoints) or `stream` (for the server sent events endpoints). If a tool never defined its own `stream` method, the dispatcher bridges `execute` into a one shot Observable automatically, so a new, simple tool never has to implement streaming itself just to satisfy the interface.

This is exactly the kind of small, reusable scaffolding a fullstack engineer should recognize as good design, a `GET /ai/tools?scope=admin` endpoint can list every registered tool's metadata for a frontend dropdown to render, without either side needing to know how many tools currently exist or what any of them actually do internally.

## `AiQueryService`, the engine behind every admin tool

Every admin scoped tool extends `BaseAdminAiTool` (`tools/admin/base-admin-tool.ts`), which just declares which subset of Claude tool names it wants exposed and what system prompt to use, then delegates the actual work to `AiQueryService.runToolConversation`. This means the two round trip Claude tool use loop, Claude decides whether to call a tool, the backend runs it, the result gets fed back for a final natural language answer, only needs to be written once.

```ts
async runToolConversation(opts: ToolConversationOptions) {
    const round1 = await this.anthropic.messages.create({ model: 'claude-sonnet-4-5', tools, messages, system: systemPrompt, ... });
    const toolUse = round1.content.find((b) => b.type === 'tool_use');
    if (!toolUse) { /* Claude answered directly, no data needed */ return { answer: text, ... }; }
    const toolResult = await this.executeTool(toolUse.name, toolUse.input);
    const round2 = await this.anthropic.messages.create({ ..., messages: [...messages, { role: 'assistant', content: round1.content }, { role: 'user', content: [{ type: 'tool_result', tool_use_id: toolUse.id, content: JSON.stringify(toolResult) }] }] });
    return { answer, data: toolResult, toolUsed: toolUse.name };
}
```

`AiQueryService`'s actual tools (`get_order_by_id`, `get_orders_by_user`, `get_orders_by_domain`, `get_user_by_email_or_id`, `get_orders_by_wallet_address`, `get_order_analytics`, and a further batch added for a general user lookup assistant, `get_user_reputation`, `get_user_builder_profile`, `get_user_nft_collections`, `get_user_contract_deployments`, `get_user_parked_domains`, `get_user_activity_summary`) reach into a wide spread of other feature modules' repositories directly, which is why `AiModule` imports so many other modules (`OrderManagementModule`, reputation, builder profile, NFT collection, contract deployment, and more). A single admin question like "who is user@example.com and what has he deployed" can end up touching five completely unrelated feature areas of the app in one Claude powered request.

`resolveUserId` is worth a close look, because it is the piece that lets an admin hand Claude a user's email, wallet address, or UUID interchangeably and have every tool resolve it consistently.

```ts
private async resolveUserId(input: { userId?: string; userEmail?: string; walletAddress?: string }): Promise<string | null> {
    if (input.userId) return input.userId;
    if (input.userEmail) {
        const normalized = normalizeEmail(input.userEmail);
        ...
        // legacy raw-lowercase fallback for emails stored before normalization existed
    }
    if (input.walletAddress) {
        const isEvmAddress = input.walletAddress.startsWith('0x') && input.walletAddress.length === 42;
        // EVM addresses matched case-insensitively; everything else matched exactly
    }
}
```

This reuses `normalizeEmail` from note 06, and its comment about a "legacy raw lowercase fallback" is a small, honest admission that the normalization rule was introduced after some real user rows had already been created without it, so the lookup has to check both forms rather than assuming every row in the database was written the same way.

`QuerySafetyGuard` (`ai/query-safety.guard.ts`) is a defensive layer that exists for a feature path that is currently commented out or unused in the actual tool implementations shown here, but its intent is clear from reading it: if an admin AI tool were ever built that let Claude generate raw SQL itself, this guard is the allowlist that would keep it from doing anything beyond a `SELECT` against a small set of named tables, explicitly blocking `tbl_user` ("contains passwords + tokens") and any statement containing `DROP`, `DELETE`, `UPDATE`, `INSERT`, or similar. It is a good example of a safety net built in before the risky feature it protects was actually finished, worth recognizing even though the tools that exist today mostly call typed repository methods instead of generating SQL directly.

## `AiInsightsService`, `AiReportService`, and `AiAlertService`, the three simpler Claude callers

Three smaller services in the AI module do not use tool use at all, they just gather some real numbers from existing repositories and ask Claude to write English about them.

`AiInsightsService.getDashboardInsights` (behind `GET /ai/insights`, cached for five minutes via `@UseInterceptors(CacheInterceptor)` and `@CacheTTL(300000)`) pulls total, daily, and monthly sales plus search summary stats from `OrderManagementRepoInterface` and `SearchManagementRepoInterface`, builds a plain text prompt asking for "a concise, professional executive summary," and hands the result back as `summary` alongside the raw `keyMetrics` so the frontend dashboard can show both the numbers and Claude's read on them side by side.

`AiReportService.generateReport` does something a little more elaborate, it asks Claude to write a full five section business report narrative, wraps that narrative in a hand built HTML template (with inline styles, since email style HTML has to be self contained), uploads the finished HTML file straight to S3 through `S3Service.uploadFile` wrapped in a fake `Express.Multer.File` object, and returns the resulting public URL. The comment on the upload step is honest about a shortcut taken here.

```ts
// Note: convert htmlContent to PDF using Puppeteer if installed.
// Fallback: upload HTML directly — browsers can print-to-PDF from it.
```

Rather than running a real HTML to PDF conversion (which would need a headless browser dependency like Puppeteer), the report ships as a plain HTML file, on the theory that any browser can already turn an HTML page into a PDF through its own print dialog, a pragmatic, if slightly manual, way to defer building real PDF generation until it is actually needed.

`AiAlertService.checkAlerts` runs on a six hour cron (`@Cron(CronExpression.EVERY_6_HOURS)`, covered again in note 05) and is meant to compare daily sales, search volume, and pending order ratios against configurable thresholds, firing LogSnag events and asking Claude for a two sentence plain English interpretation when something looks wrong. As it stands in the code today, the actual threshold checks, the LogSnag event firing, and the Claude interpretation call are all commented out, leaving `checkAlerts` returning an always empty `alerts` array. The scaffolding, the cron schedule, the in memory cache of the last result exposed through `GET /ai/alerts/status`, and the manual trigger endpoint at `POST /ai/alerts/check`, is all live and working, but the actual alerting logic inside it is currently switched off, worth noticing as a feature that is wired end to end but not yet turned back on, rather than assuming every commented block in a real codebase is simply dead.
