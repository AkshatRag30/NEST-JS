# 17. Testing in NestJS

## The two kinds of tests in this repo, and where each one lives

This project uses Jest (configured in the `jest` block of `package.json`) as its test runner. There are two distinct categories of test files here, and they answer two different questions.

Unit tests, files ending in `.spec.ts` sitting right next to the file they test inside `src` (for example `src/product/product.service.spec.ts` sits next to `src/product/product.service.ts`). These test one class in isolation, answering "does this one piece of code work correctly on its own." Run with `npm run test`.

End to end (e2e) tests, living in the separate `test` folder (`test/app.e2e-spec.ts`), configured through `test/jest-e2e.json`. These spin up the entire real application and send it real HTTP requests, answering "does the whole system actually work together, from an HTTP request all the way through to a response." Run with `npm run test:e2e`.

## Anatomy of a unit test: `product.service.spec.ts`

```ts
import { Test, TestingModule } from '@nestjs/testing';
import { ProductService } from './product.service';

describe('ProductService', () => {
  let service: ProductService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [ProductService],
    }).compile();

    service = module.get<ProductService>(ProductService);
  });

  it('should be defined', () => {
    expect(service).toBeDefined();
  });
});
```

`describe('ProductService', () => {...})` groups a set of related tests under one label, purely for organizing test output, it has no effect on behavior. `beforeEach(async () => {...})` runs before every single `it(...)` test inside this `describe` block, guaranteeing each test starts from a fresh, predictable setup rather than tests accidentally leaking state into each other.

`Test.createTestingModule({ providers: [ProductService] }).compile()` is the actual interesting part. `Test` is a utility from `@nestjs/testing` that builds a small, real, working dependency injection container, exactly like the one built for real in `main.ts` by `NestFactory.create(AppModule)`, except scoped down to only the pieces you list here, `providers: [ProductService]`. This lets you test `ProductService` with NestJS's real dependency injection machinery behind it (which matters more the moment a service has its own injected dependencies) without booting your entire application.

`module.get<ProductService>(ProductService)` asks that freshly built testing module for a real, working instance of `ProductService`, exactly like NestJS itself would hand your controller one in a live app.

`it('should be defined', () => { expect(service).toBeDefined(); })` is the actual test: it asserts that `service` is not `null` or `undefined`. This particular test is intentionally minimal, it is the default scaffold the Nest CLI generates automatically every time you create a new provider or controller with its schematics. It exists mainly to prove the wiring works (that this class can actually be constructed by the dependency injection system without errors), not to test any specific business logic.

## A test that actually checks behavior: `app.controller.spec.ts`

```ts
describe('AppController', () => {
  let appController: AppController;

  beforeEach(async () => {
    const app: TestingModule = await Test.createTestingModule({
      controllers: [AppController],
      providers: [AppService],
    }).compile();

    appController = app.get<AppController>(AppController);
  });

  describe('root', () => {
    it('should return "My Name is Farzeen Ali"', () => {
      expect(appController.getHello()).toBe('My Name is Farzeen Ali');
    });
  });
});
```

This test goes one meaningful step further than the "should be defined" scaffold. It sets up a testing module containing both `AppController` and its real dependency, `AppService`, then actually calls `appController.getHello()` and asserts the exact string it returns matches `AppService.getHello()`'s real implementation (`return 'My Name is Farzeen Ali';`, see `src/app.service.ts`). If someone later changed `AppService` to return a different string without updating this test, this exact assertion would fail, immediately telling that developer that a downstream expectation broke. This is the real value unit tests provide: they turn an assumption about your code's behavior into something that gets automatically re-checked every single time you run the test suite, catching accidental regressions before they ever reach a real user.

## A test with no dependency injection needed at all: `auth.guard.spec.ts`

```ts
import { AuthGuard } from './auth.guard';

describe('AuthGuard', () => {
  it('should be defined', () => {
    expect(new AuthGuard()).toBeDefined();
  });
});
```

Notice this one skips `Test.createTestingModule` entirely and just writes `new AuthGuard()` directly. This works because `AuthGuard` (see [11-guards-and-authorization.md](11-guards-and-authorization.md)) has an empty constructor, it has no injected dependencies of its own, so there is nothing for a dependency injection container to actually provide here. This is a useful thing to notice as a beginner: you only need `Test.createTestingModule` machinery when the class under test actually needs something injected into it. For a class with zero dependencies, plain `new ClassName()` is perfectly sufficient and simpler.

## Anatomy of the end to end test: `test/app.e2e-spec.ts`

```ts
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import * as request from 'supertest';
import { App } from 'supertest/types';
import { AppModule } from './../src/app.module';

describe('AppController (e2e)', () => {
  let app: INestApplication<App>;

  beforeEach(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    await app.init();
  });

  it('/ (GET)', () => {
    return request(app.getHttpServer())
      .get('/')
      .expect(200)
      .expect('Hello World!');
  });
});
```

This is a fundamentally bigger test than the unit tests above. `imports: [AppModule]` builds a testing module out of the entire real application, every module, controller, and provider this app has, not just one isolated class. `moduleFixture.createNestApplication()` and `await app.init()` actually boot that whole application up into a real, running (though not network-listening on a real port) Nest application instance.

`request(app.getHttpServer())` comes from Supertest, a library specifically built for testing HTTP servers. It lets you write `.get('/')`, `.expect(200)`, and `.expect('Hello World!')` to simulate an actual HTTP `GET` request to the root route and assert both the status code and the exact body of the response, exactly as a real client would experience it, but without needing an actual separate running server process or a real network call.

One detail worth flagging as a beginner trap: this e2e test expects the exact text `'Hello World!'`, but the actual current implementation in `src/app.service.ts` returns `'My Name is Farzeen Ali'`:

```ts
@Injectable()
export class AppService {
  getHello(): string {
    return 'My Name is Farzeen Ali';
  }
}
```

This means, as this repository currently stands, running `npm run test:e2e` would actually fail on this assertion, because the e2e test still expects the original Nest CLI scaffold's default text, while `AppService` was edited for this course without updating the matching e2e test to match. This is not a flaw in your understanding, it is a genuine, useful lesson: tests only protect you if they are kept in sync with the code they are testing, and a failing test like this is exactly the kind of thing a real team would catch and fix before merging a change. If you want to fix it yourself as a small exercise, updating the `.expect('Hello World!')` line in `test/app.e2e-spec.ts` to `.expect('My Name is Farzeen Ali')` would make the e2e suite pass again.

## Unit tests versus e2e tests, summarized

Unit tests are fast, numerous, and isolated, they test one class's logic directly, mocking or omitting anything that class does not strictly need. E2e tests are slower, fewer, and holistic, they test that the real, fully wired application behaves correctly from the outside, exactly the way an actual client (like a frontend app) would experience it, including all the middleware, guards, pipes, and filters actually running together in sequence, exactly the full pipeline described in [18-request-lifecycle-recap.md](18-request-lifecycle-recap.md).

## Key terms recap

Unit test: a test that verifies one class or function in isolation.

E2e (end to end) test: a test that boots the real application and verifies its behavior from an external, HTTP client's point of view.

`Test.createTestingModule({...})`: builds a scoped-down, real NestJS dependency injection container specifically for testing.

Supertest: a library for making assertions against HTTP servers in tests, without needing a real separately running server.

`describe` / `it` / `expect`: Jest's core building blocks, grouping tests, defining an individual test case, and making an assertion, respectively.
