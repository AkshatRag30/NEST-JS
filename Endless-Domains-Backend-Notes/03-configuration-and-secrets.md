# 03. Configuration and Secrets

## Why this app does not really use `.env` the way the smaller projects did

Every reference project before this one read configuration with `ConfigService` backed by a local `.env` file, and every one of them had at least one missing variable problem because that `.env` file never actually shipped. Endless Domains solves that exact problem completely differently, and understanding how is one of the most transferable, real world lessons in this whole codebase, because storing secrets in AWS Secrets Manager instead of a checked in or manually distributed file is exactly what most real companies with real infrastructure do.

## `SecretsService`, the one class everything else depends on

```ts
// src/@core/utils/secrets/secrets.service.ts
@Injectable()
export class SecretsService {
    private readonly secretsManager: SecretsManager;
    private readonly cache = new Map<string, Record<string, string>>();
    constructor() {
        this.secretsManager = new SecretsManager({ region: 'us-east-1' });
    }
    async getSecret(secretId: string): Promise<Record<string, string>> {
        if (this.cache.has(secretId)) { return this.cache.get(secretId)!; }
        const data = await this.secretsManager.getSecretValue({ SecretId: secretId });
        if (data.SecretString) {
            const parsed = JSON.parse(data.SecretString) as Record<string, string>;
            this.cache.set(secretId, parsed);
            return parsed;
        }
        return {};
    }
}
```

This class does one thing, given a secret's name (its `SecretId` inside AWS Secrets Manager), it fetches a JSON blob of key value pairs and hands them back as a plain object, exactly the shape you would otherwise get from parsing a `.env` file. The `cache` Map is what stops every single later call to `getSecret` with the same id from making a real network call to AWS every time, the very first call fetches and stores the whole bundle in memory, and every call after that, for the lifetime of the running process, is instant and free. This is the same instinct as any in memory cache, trade a small amount of staleness risk (a secret rotated in AWS mid run will not be picked up until the process restarts) for a large amount of speed and cost savings, a tradeoff worth naming explicitly since it is not commented on in the code itself.

## Where `AWS_MANAGER` fits in

`process.env.AWS_MANAGER` is, deliberately, one of the only environment variables this app still needs the traditional way, and its whole job is to be the name of the secret bundle to fetch, not a secret itself. You can see this exact pattern reused in three separate places that all need configuration before Nest's own dependency injection is fully wired up, `main.ts` fetching secrets directly before the rate limiter and AWS SDK get configured, `app.module.ts`'s `ConfigModule.forRoot({ load: [async () => { ... return secretsService.getSecret(process.env.AWS_MANAGER) ...}] })`, which is what actually populates `ConfigService` application wide, and `TypeOrmConfigService.ensureValues`, covered in the next note. Every one of those three call sites constructs its own fresh `SecretsService` and calls `getSecret` with the same secret id, meaning the underlying AWS network fetch genuinely happens more than once at startup (each `SecretsService` instance has its own separate cache), a small, real inefficiency worth noticing precisely because spotting this kind of thing, three independently constructed instances of a class that was clearly meant to be a single shared cache, is exactly the sort of improvement a fullstack engineer is expected to notice once they are comfortable enough with a codebase to look past just making a feature work.

## The Joi schema that almost validates everything

`src/app-env-validation.ts` defines a large `Joi.object({...})` schema, `POSTGRES_HOST`, `JWT_ACCESS_TOKEN_SECRET`, `STRIPE_KEY`, `CRYPTOMUS_API`, `ANTHROPIC_KEY`, and dozens more, every one of them marked `.required()` unless explicitly `.optional()`. This is the same idea as the environment validation step recommended in the LearnBridge planning notes, fail loudly at startup if something essential is missing, rather than failing confusingly later. Worth checking for yourself, by searching where `envValidationSchema` is actually imported and used, whether this schema is wired into `ConfigModule.forRoot`'s `validate` or `validationSchema` option (Nest's own supported mechanism for this) or whether it exists but is not currently plugged into the actual startup path, that distinction matters, a validation schema that is written but never executed provides no real protection at all, and is worth confirming directly in the code rather than assumed either way.

## The practical lesson for you specifically

If you ever need to add a new third party integration to this app (a new payment provider, a new blockchain RPC endpoint, anything needing an API key), the correct move is never to add it to a local `.env` file and call it done, it is to get that key added to the real AWS secret bundle this app fetches from (a conversation with whoever manages the team's AWS account, not a code change), and then read it in code exactly the way every existing module does, through `SecretsService.getSecret(process.env.AWS_MANAGER)` or through `ConfigService.get(...)` once it has been loaded in by `app.module.ts`'s `ConfigModule.forRoot`. Never hardcode a real key directly into a source file, and if you ever find one already hardcoded somewhere while exploring this codebase, that is worth flagging to a senior engineer immediately, not fixing silently, since rotating a leaked key is a coordinated action, not a solo one.
