# 09. Testing Gaps in This Repo

Every feature module in this repo has a `*.service.spec.ts` and `*.controller.spec.ts` file sitting right next to its real code, which might make it look like this project has solid test coverage. Reading them closely tells a different, more honest story, and it is a useful lesson in its own right about how NestJS's dependency injection interacts with its testing utilities.

## What every one of these spec files actually contains

Here is the entire body of `src/student/student.service.spec.ts`, and it is functionally identical (only the class names differ) to every other `.service.spec.ts` in this repo (`user`, `employee`, `product`, `library`, `project`):

```ts
import { Test, TestingModule } from '@nestjs/testing';
import { StudentService } from './student.service';

describe('StudentService', () => {
  let service: StudentService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [StudentService],
    }).compile();

    service = module.get<StudentService>(StudentService);
  });

  it('should be defined', () => {
    expect(service).toBeDefined();
  });
});
```

And every `*.controller.spec.ts` in the repo follows the exact same shape, just with `controllers: [SomeController]` instead of `providers: [SomeService]`.

This is the file the Nest CLI generates automatically the moment you run `nest generate service student` or `nest generate controller student`, before you have written a single line of real logic. At that starting point, `StudentService` had no constructor at all, so `Test.createTestingModule({ providers: [StudentService] }).compile()` genuinely had everything it needed, there is nothing else for it to resolve. The problem is that real logic was added to `StudentService` afterward, specifically the `@InjectModel(Student.name) private studentModel: Model<StudentDocument>` constructor parameter (see [04-forfeature-and-dependency-injection.md](04-forfeature-and-dependency-injection.md)), and the spec file was never updated to match.

## Why this will actually fail if you run it

`Test.createTestingModule({...}).compile()` does real dependency injection resolution, the same as `NestFactory.create()` does for the whole app. When it tries to build a `StudentService`, it sees the constructor needs something registered under the `@InjectModel(Student.name)` token, and looks for a matching provider inside the testing module it was given. Since that testing module was only told about `providers: [StudentService]`, with no `MongooseModule.forFeature([...])` and no manually supplied stand-in provider for that token, resolution fails, and `.compile()` throws an error along the lines of:

```
Nest can't resolve dependencies of the StudentService (?). Please make sure that the argument Model is available in the current context.
```

That error happens inside `beforeEach`, before the single `it('should be defined', ...)` assertion ever gets a chance to run, so the test fails outright, it never gets far enough to fail on its actual assertion.

This same problem applies to every service spec in the repo that has a constructor dependency, which is all of them except `app.service.spec.ts` is not a file that exists (there is only `app.controller.spec.ts`, and `AppService` genuinely has no constructor dependencies, so it is unaffected). It also applies to every controller spec, since every controller in this repo takes its matching service in its constructor, and every one of those `*.controller.spec.ts` files only lists `controllers: [SomeController]`, with no `providers` array supplying the service it needs, the exact same category of failure, just one layer up.

## What the fix would look like, if you wanted one

There are two standard ways to fix a spec file like this, and neither is present anywhere in this repo:

Provide a fake, hand written stand-in for the missing token, for example:

```ts
const module: TestingModule = await Test.createTestingModule({
  providers: [
    StudentService,
    {
      provide: getModelToken(Student.name),
      useValue: { find: jest.fn(), findById: jest.fn(), /* ...whatever methods the test needs */ },
    },
  ],
}).compile();
```

using `getModelToken` from `@nestjs/mongoose`, which produces exactly the same token string that `@InjectModel(Student.name)` asks for, so the container has something to hand over even though it is not a real Mongoose model.

Or, for the controller specs, provide the real (or a mocked) service directly:

```ts
const module: TestingModule = await Test.createTestingModule({
  controllers: [StudentController],
  providers: [{ provide: StudentService, useValue: { createStudent: jest.fn(), /* ... */ } }],
}).compile();
```

Neither approach appears in this repo, which is exactly why this note exists: if you run `npm run test` in this project expecting a green check mark, you are very likely to see failures, and it is worth knowing that those failures are a leftover of scaffold code that was never revisited, not a sign anything is wrong with the actual `student`, `user`, `employee`, `product`, `library`, or `project` service and controller logic itself, which is what the rest of these notes cover.

## The end to end test has a related but different problem

`test/app.e2e-spec.ts` takes a different approach, it imports the real `AppModule` wholesale rather than a hand assembled list of providers:

```ts
const moduleFixture: TestingModule = await Test.createTestingModule({
  imports: [AppModule],
}).compile();

app = moduleFixture.createNestApplication();
await app.init();
```

This avoids the missing-provider problem above entirely, since `AppModule` really does import every feature module with its real `MongooseModule.forFeature(...)` registrations. But it introduces a different dependency: `AppModule` also calls `MongooseModule.forRoot(process.env.MONGO_URI!)` (see [02-connecting-to-mongodb.md](02-connecting-to-mongodb.md)), and `app.init()` will actually attempt a real network connection to MongoDB at that point. Since this repo ships no `.env` file, `MONGO_URI` will be `undefined` in any environment that has not had one created locally, and this test will fail before it ever gets to its actual `/ (GET)` assertion, either because `mongoose.connect(undefined)` throws immediately, or because there is simply no real database reachable at whatever URI was supplied. Running this end to end test successfully is only possible once you provide your own working `MONGO_URI`, which is a reasonable thing to expect of anyone actually deploying this app, but is worth knowing up front rather than being confused by an unexplained failure.
