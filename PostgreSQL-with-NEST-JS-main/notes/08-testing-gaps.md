# 08. Testing Gaps in This Repo

This repo has five `.spec.ts` files, one for the root app controller, two for the employees feature, one for the Supabase guard, and one end to end test. Reading every one of them closely tells a similar story to the MongoDB sibling project's testing gaps note, several of these were never updated after real constructor dependencies were added to the classes they test, but this repo also has one spec file with a more serious problem than anything in the sibling project, a genuine TypeScript compile error.

## `app.controller.spec.ts`, the one file that actually passes

```ts
const app: TestingModule = await Test.createTestingModule({
  controllers: [AppController],
  providers: [AppService],
}).compile();
```

`AppController` needs an `AppService` in its constructor, and `AppService` itself has no constructor dependencies at all, it is the plain, untouched Nest CLI starter service. This testing module supplies exactly what is needed, `AppService` is listed as a provider, so `.compile()` succeeds, and the single assertion, `expect(appController.getHello()).toBe('Hello World!')`, genuinely passes.

## `employees.service.spec.ts`, missing the repository provider

```ts
const module: TestingModule = await Test.createTestingModule({
  providers: [EmployeesService],
}).compile();
```

`EmployeesService`'s constructor needs `@InjectRepository(Employee) private employeeRepository: Repository<Employee>`, covered in [04-employees-module-repository-and-service.md](04-employees-module-repository-and-service.md). This testing module only lists `EmployeesService` itself, with no `TypeOrmModule.forFeature([Employee])` and no manually supplied stand in for the repository token. When `.compile()` tries to build `EmployeesService`, it looks for a provider registered under the token TypeORM uses internally for `Repository<Employee>` (produced by `getRepositoryToken(Employee)` from `@nestjs/typeorm`), finds nothing, and throws a dependency resolution error before the single `it('should be defined', ...)` assertion ever runs. This is exactly the same category of failure the MongoDB sibling project's testing gaps note describes for its own service specs, generated scaffold code that was never revisited once a real constructor dependency was added.

## `employees.controller.spec.ts`, missing the service provider

```ts
const module: TestingModule = await Test.createTestingModule({
  controllers: [EmployeesController],
}).compile();
```

Same problem, one layer up. `EmployeesController`'s constructor needs an `EmployeesService`, and this testing module never supplies one, no `providers` array at all. `.compile()` fails to resolve `EmployeesController`'s constructor for the same reason as above, before its own `it('should be defined', ...)` assertion gets a chance to run.

## `supabase-auth.guard.spec.ts`, a more serious problem than a missing provider

```ts
import { SupabaseAuthGuard } from './supabase-auth.guard';

describe('SupabaseAuthGuard', () => {
  it('should be defined', () => {
    expect(new SupabaseAuthGuard()).toBeDefined();
  });
});
```

This one is worth reading even more carefully, because it fails differently, and earlier, than the two above. It does not use `Test.createTestingModule()` at all, it constructs `SupabaseAuthGuard` directly with a bare `new SupabaseAuthGuard()`. But `SupabaseAuthGuard`'s constructor is `constructor(private configService: ConfigService){}`, one required parameter with no default value and no `?` marking it optional, covered in [07-supabase-auth-guard.md](07-supabase-auth-guard.md). Calling `new SupabaseAuthGuard()` with zero arguments against a constructor that requires exactly one is a TypeScript type error, "Expected 1 arguments, but got 0", the kind of error the compiler catches regardless of how strict the rest of `tsconfig.json` is configured, since checking the number and type of arguments passed to a constructor is basic type checking, not a `strict` mode only feature.

Since this project's Jest configuration transforms `.ts` files through `ts-jest` (see `package.json`'s `jest` block, `"transform": { "^.+\\.(t|j)s$": "ts-jest" }`) with no `isolatedModules` flag turned on to skip type checking, `ts-jest` performs full type checking on this file before it ever runs. That means this spec file would fail before Jest even gets to execute the `describe`/`it` blocks, it fails at the compile step itself, with a TypeScript error pointing at that exact `new SupabaseAuthGuard()` call. This is a more fundamental failure than the missing provider problem in the two employees specs above, those two are syntactically and structurally valid TypeScript that fail only when NestJS's dependency injection container actually tries to resolve a constructor at runtime, this one does not even get that far, it never becomes valid, runnable JavaScript in the first place.

## What the fix would look like, for all three broken specs

For the two employees specs, the same two standard fixes from the sibling project's testing notes apply here. Either supply a hand written stand in for the missing token:

```ts
import { getRepositoryToken } from '@nestjs/typeorm';

const module: TestingModule = await Test.createTestingModule({
  providers: [
    EmployeesService,
    { provide: getRepositoryToken(Employee), useValue: { find: jest.fn(), findOneBy: jest.fn() } },
  ],
}).compile();
```

or, for the controller spec, supply a mocked service directly:

```ts
const module: TestingModule = await Test.createTestingModule({
  controllers: [EmployeesController],
  providers: [{ provide: EmployeesService, useValue: { findAll: jest.fn() } }],
}).compile();
```

For the guard spec, the fix is more basic, it needs to actually construct the class correctly, either by passing a real or mocked `ConfigService` directly, `new SupabaseAuthGuard({ get: () => 'some-secret' } as any)`, or by going through `Test.createTestingModule({ providers: [SupabaseAuthGuard, ConfigService] }).compile()` the way NestJS testing normally works. None of these fixes appear anywhere in this repo, all three spec files are exactly as generated or written, unmodified since the real dependencies were added to the classes they claim to test.

## The end to end test, a different problem again

```ts
const moduleFixture: TestingModule = await Test.createTestingModule({
  imports: [AppModule],
}).compile();

app = moduleFixture.createNestApplication();
await app.init();
```

`test/app.e2e-spec.ts` imports the real `AppModule` wholesale, which sidesteps the missing provider problem entirely, `AppModule` really does wire up `EmployeesModule` and `UserModule` with their real `TypeOrmModule.forFeature([...])` registrations. But `AppModule` also calls `TypeOrmModule.forRoot({ url: process.env.DATABASE_URL, ... })`, covered in [01-project-setup-and-postgres-connection.md](01-project-setup-and-postgres-connection.md), and `app.init()` is the exact moment that connection actually gets attempted. Since this repo ships no `.env` file, `DATABASE_URL` is `undefined` in any environment that has not had one created locally, and this test will fail before it ever reaches its `/ (GET)` assertion, because there is no real, reachable Postgres database for TypeORM to connect to. Running this end to end test successfully requires supplying your own working `DATABASE_URL` first, the same category of requirement the MongoDB sibling project's own e2e test has around `MONGO_URI`.

## The honest summary

Out of five spec files in this repo, only `app.controller.spec.ts` would actually pass as written. The two employees specs fail at the dependency injection resolution step inside `.compile()`. The Supabase guard spec fails even earlier, at TypeScript compilation, before Jest can run anything in it at all. And the one end to end test depends entirely on a real Postgres database being reachable through an environment variable this repo does not, and by design should not, ship with a real value for.
