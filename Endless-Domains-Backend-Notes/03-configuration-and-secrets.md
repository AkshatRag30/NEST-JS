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

## Update from the October 2026 uat pull

Between commit `a131b429` and `uat` merge commit `dc1ba3e8`, the AWS secret bundle this app depends on grew by more than twenty keys, almost all for the new marketplace. The way they are read is a good, concrete example of everything this note explains.

### The question this note asked you to check, answered

The section on the Joi schema above asks you to find out whether `envValidationSchema` is actually wired into `ConfigModule.forRoot`. It is not. In `src/app.module.ts` the relevant lines are commented out:

```ts
// src/app.module.ts
        ConfigModule.forRoot({
            isGlobal: true,
            load: [async () => {
                const secretsService = new SecretsService();
                const secrets = await secretsService.getSecret(process.env.AWS_MANAGER);
                return {
                    ...secrets,
                };
            }]
            // envFilePath: '.env',
            // validationSchema: envValidationSchema
        }),
```

So the Joi schema in `src/app-env-validation.ts` provides no protection at startup today, and none of the new marketplace keys were added to it anyway. Marketplace v2 handles this differently, and better. It brings its own validated loader, which is worth reading as the model for how config validation should look.

### Marketplace v2's own validated config loader

`src/components/marketplacev2/config/chain-config.loader.ts` fetches the same `AWS_MANAGER` secret through `SecretsService`, picks out its own keys, runs each one through a small validator (`chain-config.validators.ts`: `validateChecksumAddress` normalizes every address through `ethers.getAddress` and rejects anything invalid, `validateRpcUrl` only requires that `new URL(value)` parses, so any scheme is accepted, and it deliberately never echoes the value because RPC URLs usually contain an API key, `validateFeeBps` requires an integer from 1 to 10000, and `validateStartBlock` requires a positive integer), and throws a `ChainConfigError` (from `chain-config.errors.ts`) naming the bad key if anything is wrong. The result is assembled into a typed `ChainConfig` object and provided through an injection token, so no other code in the marketplace ever calls `ConfigService.get('SOME_STRING')` directly. That is three improvements over the older pattern in one place: validation runs, the error names the key, and consumers get a typed object instead of `string | undefined`. The cost, covered in the root architecture note, is that a bad marketplace key now stops the entire API from booting. Full detail is in [clusters/12-marketplace-v2/01-what-marketplace-v2-is-and-the-module-map.md](clusters/12-marketplace-v2/01-what-marketplace-v2-is-and-the-module-map.md).

### Every new key, with the names the code actually reads

| Key in the AWS secret | Required | What it is |
|---|---|---|
| `POL_RPC_URL` | yes | Polygon JSON RPC endpoint used by order validation, the poller and the expiry sweep |
| `SEAPORT_ADDRESS_POLYGON` | yes | the Seaport contract that fills orders |
| `USDT_ADDRESS_POLYGON` | yes | the payment token every v2 order is priced in |
| `DOMAIN_NFT_ADDRESS_POLYGON_UD` | yes | the domain NFT collection that can be listed (Unstoppable Domains on Polygon) |
| `FEE_RECIPIENT` | yes | the address that receives the marketplace fee |
| `FEE_BPS` | yes | the marketplace fee in basis points, an integer from 1 to 10000 |
| `POLLER_START_BLOCK_POLYGON` | no | first block the poller reads; falls back to 93,840,000 with only a console warning |
| `POLLER_ENABLED` | no | turns the poller and expiry sweep on; falls back to `NODE_ENV === 'production'` |
| `MARKET_DATA_ENABLED` | no | turns the OpenSea jobs on; read on every tick, but the secret is cached forever, so a change in practice needs a restart |
| `OPENSEA_API_KEY` | when market data is on | sent as the `x-api-key` header; boot fails if the flag is on and this is missing |
| `COINGECKO_API_KEY` | no | sent as `x-cg-demo-api-key` for USD prices |
| `MARKET_ENS_ETH_BASE_REGISTRAR_CONTRACT`, `MARKET_ENS_ETH_NAME_WRAPPER_CONTRACT`, `MARKET_UD_POL_CONTRACT`, `MARKET_UD_BASE_CONTRACT`, `MARKET_SPACEID_BNB_CONTRACT`, `MARKET_SPACEID_ARB_CONTRACT`, `MARKET_FREENAME_POL_CONTRACT`, `MARKET_FREENAME_BNB_CONTRACT`, `MARKET_FREENAME_BASE_CONTRACT`, `MARKET_UD_ETH_CONTRACT` | per collection, the last one optional | the collections OpenSea stats are pulled for; a missing one only drops that collection |

The chain id, 137 for Polygon, is a hardcoded constant rather than a key.

### The `.env.sample` trap

`.env.sample` gained a commented block for these keys in this update, and it is wrong in three ways that would cost a new developer an afternoon. It uses different names (`POLYGON_RPC_URL`, `SEAPORT_ADDRESS`, `USDT_ADDRESS`, `DOMAIN_NFT_ADDRESS`) from the ones the code reads. It leaves out the two `POLLER_*` keys. And it shows a `FEE_BPS` of zero, which the validator rejects. Its own comment does say, correctly, that these values are not read from `process.env` and must go into the `AWS_MANAGER` secret. The same file still shows `JWT_ACCESS_TOKEN_EXPIRATION=3d`, while a comment in `require-verified-wallet.guard.ts` records that the real secret serves `1h`. The general lesson from this note holds even more strongly now: a sample file is documentation, and documentation drifts. The loader code is the truth.

### Caching of secrets now matters at runtime

This note mentioned that `SecretsService` caches the bundle for the life of the process. That used to be only a startup detail. Now several values behave like runtime switches, `MARKET_DATA_ENABLED` above all, and their code reads them "every tick". Because the bundle is cached forever, flipping the value in AWS still does nothing until the process restarts. The region is also still hardcoded to `us-east-1` inside `SecretsService`.
