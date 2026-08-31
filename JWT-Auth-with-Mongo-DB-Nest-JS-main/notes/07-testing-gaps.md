# 07. Testing Gaps in This Repo

This project has four `.spec.ts` files under `src`, plus one end to end test under `test`. Reading each one closely, and thinking through what actually happens when `Test.createTestingModule({...}).compile()` runs, shows that none of the four unit spec files will pass if you actually run `npm run test`, and each fails for a genuinely different reason. This is the same kind of honest gap documented in the sibling MongoDB project's notes, worth working through in full here rather than assuming it away.

## `app.controller.spec.ts`, the one file that actually passes

```ts
const app: TestingModule = await Test.createTestingModule({
  controllers: [AppController],
  providers: [AppService],
}).compile();
```

`AppController` needs an `AppService` in its constructor, and `AppService` itself has no constructor dependencies at all, it is the plain, unmodified Nest CLI starter service. This testing module supplies exactly what is needed, nothing is missing, so `.compile()` succeeds and the single assertion, `expect(appController.getHello()).toBe('Hello World!')`, genuinely passes. This file is untouched scaffold code, and it happens to still be correct because nothing was ever added to `AppController` or `AppService` that would require anything more.

## `auth.controller.spec.ts`, missing the service it depends on

```ts
const module: TestingModule = await Test.createTestingModule({
  controllers: [AuthController],
}).compile();
```

`AuthController`'s constructor is `constructor(private authService: AuthService){}`, it needs a real `AuthService` (or a stand in for one) to be constructed. This testing module only lists `controllers: [AuthController]`, with no `providers` array supplying `AuthService` at all. When `.compile()` tries to build `AuthController`, it cannot find anything registered for the `AuthService` token, and dependency injection resolution fails with an error along the lines of `Nest can't resolve dependencies of the AuthController (?). Please make sure that the argument AuthService is available in the current context.` That failure happens inside `beforeEach`, before the single `it('should be defined', ...)` assertion ever runs, so the test fails outright.

## `auth.service.spec.ts`, missing two dependencies at once

```ts
const module: TestingModule = await Test.createTestingModule({
  providers: [AuthService],
}).compile();
```

This one is a step further than the previous file, because `AuthService`'s constructor needs not one but two things it does not get here: `@InjectModel(User.name) private userModel: Model<UserDocument>`, and `private jwtService: JwtService`. Providing `providers: [AuthService]` alone gives the testing module no provider registered under the `getModelToken('User')` token that `@InjectModel(User.name)` asks for, and no `JwtService` provider either, since that normally only becomes available by importing `JwtModule` (as `AuthModule` does in the real app). Dependency injection resolution will fail on whichever of the two missing dependencies it attempts to resolve first, with the same category of "Nest can't resolve dependencies" error as above. This test also fails inside `beforeEach`, before its own `it('should be defined', ...)` assertion has any chance to run.

## `user.schema.spec.ts`, a different and more basic kind of broken

```ts
import { UserSchema } from './user.schema';

describe('UserSchema', () => {
  it('should be defined', () => {
    expect(new UserSchema()).toBeDefined();
  });
});
```

This file fails for a reason that has nothing to do with dependency injection at all, and it is worth being precise about why, since it is a different category of mistake from the two above. `UserSchema`, exported from `user.schema.ts`, is the result of `SchemaFactory.createForClass(User)`, and that call returns an actual instance of Mongoose's `Schema` class, an object, already fully constructed. It is not a class or a constructor function itself, it is the finished product of one. Writing `new UserSchema()` asks JavaScript to treat `UserSchema` as something callable with `new`, which it is not, since it is a plain object value rather than a function or class. Running this file throws a `TypeError` along the lines of `UserSchema is not a constructor`, immediately, before `.toBeDefined()` ever gets a chance to run. This test is not failing because a dependency is missing, it is failing because the test itself does not reflect what `UserSchema` actually is, most likely because it was scaffolded by copying the same "should be defined" shape used for a controller or service spec (where `new SomeController()` or `new SomeService()` genuinely would make sense, since those are real classes) and pointing it at a schema constant without adjusting for what a compiled Mongoose schema actually is. A test that would have actually exercised this file meaningfully might instead check `UserSchema.obj` or use `SchemaFactory.createForClass(User)` again directly to confirm the expected paths exist, or exercise `User` the plain class itself, but nothing like that is present here.

## What the fix would look like, if you wanted one

For `auth.controller.spec.ts`, supplying a mock service directly would let it compile without needing a real MongoDB connection or a real JWT secret:

```ts
const module: TestingModule = await Test.createTestingModule({
  controllers: [AuthController],
  providers: [{ provide: AuthService, useValue: { signup: jest.fn(), login: jest.fn() } }],
}).compile();
```

For `auth.service.spec.ts`, the same idea applies to both missing dependencies, using `getModelToken` from `@nestjs/mongoose` for the model token and a plain mock object for `JwtService`:

```ts
const module: TestingModule = await Test.createTestingModule({
  providers: [
    AuthService,
    { provide: getModelToken(User.name), useValue: { findOne: jest.fn() } },
    { provide: JwtService, useValue: { sign: jest.fn() } },
  ],
}).compile();
```

For `user.schema.spec.ts`, the fix is simply to stop calling `new` on `UserSchema` and instead assert something true about the schema object itself, for example `expect(UserSchema).toBeDefined()` with no `new`, or inspecting `UserSchema.obj.email` to confirm the expected fields were registered.

None of these fixes are present anywhere in this repository, which is exactly why this note exists: running `npm run test` on this project as it stands will very likely produce failures across three of its four spec files, and it is worth understanding that those failures are leftover scaffold code and a small misunderstanding of what a compiled schema is, not a sign that the real signup, login, hashing, or token verification logic covered in the earlier notes is broken. That logic is genuinely correct and does work when actually run through the app.

## The end to end test needs two things this repo does not ship

```ts
const moduleFixture: TestingModule = await Test.createTestingModule({
  imports: [AppModule],
}).compile();

app = moduleFixture.createNestApplication();
await app.init();
```

`test/app.e2e-spec.ts` sidesteps the missing-provider problem entirely, since it imports the real `AppModule`, which really does wire up every real provider `AuthController` and `AuthService` need, through the real `AuthModule`. But this introduces two different, environment level dependencies instead. `AppModule` calls `MongooseModule.forRoot(process.env.MONGO_URI!)`, so `app.init()` will attempt a genuine network connection to MongoDB, which fails if `MONGO_URI` is not set, or is not a real reachable database. And, as covered in [05](05-passport-jwt-strategy-and-protected-routes.md), `AuthModule` also eagerly constructs `JwtStrategy`, whose constructor throws immediately if `MY_JWT_SECRET` is not set anywhere in the environment. Since neither variable ships with this repository, in any environment that has not had both configured, this end to end test will fail before its actual `/ (GET)` assertion ever runs, for either or both of these reasons. This is a reasonable thing to expect of anyone actually deploying this app (you do need a real database and a real secret to run a real authentication service), but it is worth knowing up front rather than being confused by an unexplained failure the first time this test is run.
