# 05. Testing, a Real Bug in Both Spec Files, and a Full Recap

## The unit test: `src/app.controller.spec.ts`

```ts
import { Test, TestingModule } from '@nestjs/testing';
import { AppController } from './app.controller';
import { AppService } from './app.service';

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
    it('should return "Hello World!"', () => {
      expect(appController.getHello()).toBe('Hello World!');
    });
  });
});
```

Start with whether this test can even be constructed. `Test.createTestingModule({ controllers: [AppController], providers: [AppService] }).compile()` builds a small, real dependency injection container containing exactly those two classes. `AppController`'s constructor needs one thing, `private readonly appService: AppService`, and `AppService` is right there in the `providers` array. `AppService` itself, as you can see in `src/app.service.ts`, has no constructor of its own, so it needs nothing further to be built. This means dependency injection resolution succeeds cleanly here, unlike the kind of missing provider failure you would see if a service needed something like an injected database model that the test never supplied. So far, this test is set up correctly.

## Where it actually goes wrong

The test calls `appController.getHello()` and asserts the result equals the exact string `'Hello World!'`. Now compare that against what `getHello()` in the real `app.controller.ts` actually does:

```ts
@Get()
@Throttle({ default: { limit: 3, ttl: 60000} })
getHello(): string {
  return 'This is a rate-limit route!'
}
```

The method does not call `this.appService.getHello()` at all. It returns a hardcoded literal string directly, `'This is a rate-limit route!'`. The string `'Hello World!'` that the test expects is actually what `AppService.getHello()` returns, in `src/app.service.ts`:

```ts
@Injectable()
export class AppService {
  getHello(): string {
    return 'Hello World!';
  }
}
```

But `AppController.getHello()` never invokes that method. This means `appController.getHello()` really returns `'This is a rate-limit route!'`, not `'Hello World!'`, and the assertion `expect(appController.getHello()).toBe('Hello World!')` will fail the moment you actually run `npm run test`. This is a genuine bug in this repository's test suite as it stands, not a misunderstanding on a beginner's part, if you run the tests you will see a real, red failure here.

## A second, related gap: an injected service that is never used

Look again at the constructor, `constructor(private readonly appService: AppService) {}`. `AppService` is injected into `AppController` exactly the way you would expect from the standard Nest CLI scaffold, where a freshly generated controller's `getHello()` normally does `return this.appService.getHello();`. But in this project, `getHello()` was rewritten to return its own hardcoded, rate limit specific message instead, and the injected `appService` field is left completely unused inside the class body. It still gets constructed and injected on every request, it is just never read. This is not something that breaks anything at runtime, TypeScript does not flag it here since the parameter is captured as a class property, but it is a real, honest sign of incomplete cleanup, the demo message was swapped in without either deleting the now pointless `AppService` dependency or actually wiring it into the response, and this is exactly the kind of small gap that caused the spec file above to still expect the old text.

## The end to end test has the exact same problem

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

This test takes the more thorough approach of importing the entire real `AppModule`, so the whole application, including the global `ThrottlerGuard` registered through `APP_GUARD` (see [03-global-guards-and-app-guard.md](03-global-guards-and-app-guard.md)), actually boots up for this test, exactly as it would in production. `moduleFixture.createNestApplication()` and `app.init()` bring the real server to life, and `request(app.getHttpServer()).get('/')` uses Supertest to send one real `GET` request to the root route. Since this test only sends a single request, it stays comfortably under the configured limit of three requests per sixty seconds, so throttling itself does not interfere with this test running.

But the final assertion, `.expect('Hello World!')`, checks the response body against the same string the unit test expected, and for the exact same reason it fails there, it will fail here too, the real running server responds with `'This is a rate-limit route!'`, not `'Hello World!'`. So both the unit test and the end to end test in this repository fail on the same underlying mismatch, one small hardcoded string that was changed in `app.controller.ts` for the rate limiting demo without the two test files being updated to match.

## What fixing it would actually look like

There are two equally valid ways to fix this, and neither is present in the repository as it stands. One option is to change the two `.toBe('Hello World!')` and `.expect('Hello World!')` lines in `src/app.controller.spec.ts` and `test/app.e2e-spec.ts` to expect `'This is a rate-limit route!'` instead, matching what the controller genuinely returns. The other option is to change `app.controller.ts` itself so `getHello()` actually returns `this.appService.getHello()`, putting the injected `AppService` back to real use, and then decide what that service should return, whether that is `'Hello World!'` or something rate limiting specific. Either fix is a one line change, but as the code is currently written, running `npm run test` or `npm run test:e2e` in this project will not give you a clean, green result.

## The gap that matters most: throttling itself is never tested

Step back and notice what neither test actually checks. This entire project exists to demonstrate rate limiting, yet nowhere in `src/app.controller.spec.ts` or `test/app.e2e-spec.ts` does any test send more than one request, and no test ever asserts that a fourth request within the sixty second window gets rejected with a `429` status and the custom `errorMessage` configured in `app.module.ts`. A real end to end test for this project's actual purpose would need to send at least four requests in quick succession to the same route and check that the first three succeed while the fourth comes back with a `429`. That test simply does not exist here. This is worth calling out plainly, the one thing this whole repository is built to show off is also the one thing its test suite never actually verifies.

## Recap: the life of one request through this app

Put everything from these five notes together, and here is exactly what happens when a client sends a `GET /` request to this running application. The request first reaches `ThrottlerGuard`, which runs on every route in the app because it was registered globally through the `APP_GUARD` token in `app.module.ts` (see [03-global-guards-and-app-guard.md](03-global-guards-and-app-guard.md)), with no `@UseGuards()` decorator needed anywhere. `ThrottlerGuard` looks at this specific route and sees it carries its own `@Throttle({ default: { limit: 3, ttl: 60000 } })` decorator (see [04-per-route-throttle-overrides.md](04-per-route-throttle-overrides.md)), so it checks this client's request count for the `'default'` profile against those numbers, three allowed requests per sixty second window, which happen to be numerically identical to the module's own global default. If this client has made fewer than three requests within the current window, the request is allowed through and `AppController.getHello()` runs, returning the hardcoded string `'This is a rate-limit route!'` (never touching the injected but unused `AppService`). If this is the client's fourth request within that same sixty second window, `ThrottlerGuard` rejects it immediately, the controller method never runs at all, and the client receives an HTTP `429 Too Many Requests` response carrying the custom message configured back in `app.module.ts`, `'Too many requests! Please wait a minute and try again!'`. That single flow, one guard, one module level configuration, one optional per route override, is the entirety of what this project teaches, and now you have seen every piece of it, including the two places where its own tests do not yet match its own code.
