# 08. Testing Gaps in This Repo

This project has four spec files in total, `src/app.controller.spec.ts`, `src/book/book.service.spec.ts`, `src/book/resolvers/book.resolver.spec.ts`, and `test/app.e2e-spec.ts`. Reading all four closely turns up exactly the same category of problem the MongoDB sibling project's own testing notes describe: unmodified Nest CLI scaffold tests that never got updated after real constructor dependencies were added to the classes they test. This note goes through each one honestly.

## book.service.spec.ts will fail

```ts
import { Test, TestingModule } from '@nestjs/testing';
import { BookService } from './book.service';

describe('BookService', () => {
  let service: BookService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [BookService],
    }).compile();

    service = module.get<BookService>(BookService);
  });

  it('should be defined', () => {
    expect(service).toBeDefined();
  });
});
```

This is the exact file the Nest CLI generates automatically the moment you run `nest generate service book`, before any real logic exists yet. At that starting point `BookService` had no constructor at all, so `providers: [BookService]` genuinely had everything `.compile()` needed. Real logic was added afterward, specifically the constructor parameter `@InjectModel(Book.name) private bookModel: Model<Book>` (covered in [05-the-book-service.md](05-the-book-service.md)), and this spec file was never touched again to match. `Test.createTestingModule({...}).compile()` performs real dependency injection resolution, and when it tries to build `BookService` it needs something registered under the `@InjectModel(Book.name)` token, which nothing in `providers: [BookService]` supplies. `.compile()` will throw an error along the lines of:

```
Nest can't resolve dependencies of the BookService (?). Please make sure that the argument Model is available in the current context.
```

That failure happens inside `beforeEach`, before the single `it('should be defined', ...)` assertion ever runs, so this test fails outright every time, regardless of anything correct or incorrect about `BookService`'s own logic.

## book.resolver.spec.ts will fail, for the same underlying reason one layer up

```ts
import { Test, TestingModule } from '@nestjs/testing';
import { BookResolver } from './book.resolver';

describe('BookResolver', () => {
  let resolver: BookResolver;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [BookResolver],
    }).compile();

    resolver = module.get<BookResolver>(BookResolver);
  });

  it('should be defined', () => {
    expect(resolver).toBeDefined();
  });
});
```

Same shape, same problem, one layer higher up the call chain. `BookResolver`'s constructor needs a real `BookService` instance (`constructor(private readonly bookService: BookService){}`), and this testing module only lists `providers: [BookResolver]`, nothing supplying `BookService` itself. Since `BookService` cannot be constructed without its own `Model<Book>` dependency either, resolving it here would hit the exact same missing dependency problem described above, one level removed. This test fails in `beforeEach` for the same underlying reason, a scaffold file generated before the resolver had a real constructor dependency, never revisited once one was added.

## app.controller.spec.ts is fine, and it is worth seeing why

```ts
const app: TestingModule = await Test.createTestingModule({
  controllers: [AppController],
  providers: [AppService],
}).compile();
```

This one passes, and the reason is instructive by contrast. `AppController`'s constructor needs an `AppService`, and this testing module actually supplies one, `providers: [AppService]`. `AppService` itself has no constructor parameters at all (`getHello(): string { return 'Hello World!'; }` is its entire body), so there is nothing further for the container to resolve. This is the untouched, still accurate Nest CLI scaffold test for the one class in this project simple enough that the scaffold never went stale, and it is a useful reference for what the `book` module's two broken spec files would need to look like to actually pass: either supply a real or fake `BookService` in `book.resolver.spec.ts`'s `providers` array, or supply a fake stand in for the `@InjectModel(Book.name)` token, using `getModelToken` from `@nestjs/mongoose`, in `book.service.spec.ts`'s `providers` array, for example:

```ts
const module: TestingModule = await Test.createTestingModule({
  providers: [
    BookService,
    {
      provide: getModelToken(Book.name),
      useValue: { find: jest.fn(), findById: jest.fn(), findByIdAndDelete: jest.fn() },
    },
  ],
}).compile();
```

Neither fix appears anywhere in this repository today.

## test/app.e2e-spec.ts: a different problem, needing a real database

```ts
const moduleFixture: TestingModule = await Test.createTestingModule({
  imports: [AppModule],
}).compile();

app = moduleFixture.createNestApplication();
await app.init();

it('/ (GET)', () => {
  return request(app.getHttpServer())
    .get('/')
    .expect(200)
    .expect('Hello World!');
});
```

This test avoids the missing provider problem entirely, since it imports the real `AppModule` wholesale, which really does register `BookModule` with its genuine `MongooseModule.forFeature(...)` binding intact, so `BookService` and `BookResolver` would both resolve correctly here if anything in this test actually exercised them (it does not, the one assertion only hits the plain REST `GET /` route). But `AppModule` also runs `MongooseModule.forRoot(process.env.MONGO_URI!)` at the top level, and `app.init()` is the exact moment NestJS actually attempts that real MongoDB connection. As covered in [01-project-setup-and-graphql-module.md](01-project-setup-and-graphql-module.md), this repository ships no `.env` file at all, confirmed by its absence from every file in this project and by `.gitignore` explicitly excluding it, so `MONGO_URI` is `undefined` in any environment that has not had one created locally. Running `npm run test:e2e` in a fresh checkout of this repository, with no `.env` file added and no `MONGO_URI` otherwise set, would fail before ever reaching the `/ (GET)` assertion, either because Mongoose rejects an undefined connection string outright or because there is simply no reachable database at whatever value ends up being used. This is a real, checkable gap, not a hypothetical one, and it is the same category of problem the MongoDB sibling project's own end to end test has, for the same underlying reason, a required environment variable this repository never ships a default or example value for.

## The honest summary

If you clone this repository and run `npm run test` expecting every check to pass, it will not, `book.service.spec.ts` and `book.resolver.spec.ts` both fail in `beforeEach`, before their one assertion each ever runs, and `npm run test:e2e` fails too unless you first create your own working `MONGO_URI`. None of this reflects a bug in `BookService`, `BookResolver`, or the resolver's actual GraphQL behavior, which the rest of these notes cover and which work correctly once the app is actually running against a real database. It reflects scaffold test files generated at the very start of the project, before real constructor dependencies existed, that were never updated once those dependencies were added, exactly the same lesson the MongoDB sibling project's own testing notes draw from its own six feature modules.
