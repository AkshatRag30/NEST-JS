# 09. Full Recap: One Request, Start to Finish

This last note traces one complete HTTP request through every layer covered in this folder, and then gives you a single table mapping every file in this repo to the note that explains it, so this folder can double as a quick reference after your first read through.

## Tracing `GET /employees`, from HTTP request to Postgres and back

This is the one route in the whole app that touches every major concept in this folder at once, since it is the only route protected by the auth guard.

Everything starts at app boot. `main.ts` calls `NestFactory.create(AppModule)`, which reads `AppModule`'s imports and builds the whole dependency injection container in order: `ConfigModule.forRoot({ isGlobal: true })` loads any `.env` file into a form `ConfigService` can read, `TypeOrmModule.forRoot({ type: 'postgres', url: process.env.DATABASE_URL, autoLoadEntities: true, synchronize: true })` opens the one shared connection to Postgres (assuming a real `DATABASE_URL` has actually been supplied locally, see [01-project-setup-and-postgres-connection.md](01-project-setup-and-postgres-connection.md)), and `EmployeesModule` is imported, which itself calls `TypeOrmModule.forFeature([Employee])`, registering an injectable `Repository<Employee>` tied to that shared connection.

A client sends `GET /employees` with a header `Authorization: Bearer <some jwt>`. Nest's router matches this to `EmployeesController.findAll`, because `@Controller('employees')` combined with the bare `@Get()` on that method is exactly `GET /employees`. Before `findAll` itself runs, though, `@UseGuards(SupabaseAuthGuard)` on that same method means `SupabaseAuthGuard.canActivate()` runs first. The guard pulls the `Authorization` header off the request, confirms it starts with `Bearer `, strips off the token itself, reads `SUPABASE_JWT_SECRET` through its injected `ConfigService`, and calls `jwt.verify(token, jwtSecret)` entirely locally, no network call to Supabase happens here, covered fully in [07-supabase-auth-guard.md](07-supabase-auth-guard.md). If verification succeeds, the decoded payload is attached to the request as `request['user']`, and the guard returns `true`, letting the request proceed. If the header is missing, malformed, or the token fails verification, the guard throws `UnauthorizedException` and `findAll` never runs at all.

Assuming the guard passes, `EmployeesController`'s constructor already has a working `EmployeesService` instance in hand, injected automatically when the app started, thanks to `EmployeesModule` listing `EmployeesService` in its `providers` array. `findAll()` does nothing but delegate: `return this.employeeService.findAll();`.

Inside `EmployeesService.findAll()`, `this.employeeRepository`, injected via `@InjectRepository(Employee)` (see [04-employees-module-repository-and-service.md](04-employees-module-repository-and-service.md)), is a TypeORM `Repository<Employee>` tied to the `employee` table, made possible because `EmployeesModule` registered that entity with `TypeOrmModule.forFeature([Employee])`, and `Employee` itself was defined earlier using `@Entity()`/`@PrimaryGeneratedColumn()`/`@Column()` in `employees.entity.ts` (see [03-typeorm-entities-vs-mongoose-schemas.md](03-typeorm-entities-vs-mongoose-schemas.md)). `this.employeeRepository.find()` issues a plain `SELECT * FROM employee` style query against Postgres and resolves with every row, mapped back into typed `Employee` objects.

That array of `Employee` objects travels back up through `EmployeesService` to `EmployeesController`, and Nest's built in response handling serializes whatever the controller method returns into a JSON HTTP response automatically, no manual `res.json(...)` call needed anywhere in this codebase.

If the same client instead called `POST /employees`, `GET /employees/search`, `GET /employees/:id`, `PUT /employees/:id`, or `DELETE /employees/:id`, the exact same dependency injected `EmployeesService` and `Repository<Employee>` would be used underneath, but the request would reach the controller method directly, with no guard check at all, since `findAll` is the only method on this controller carrying `@UseGuards(SupabaseAuthGuard)`, see [05-employees-controller-routes.md](05-employees-controller-routes.md).

## File map

| File | What it teaches | Note |
|---|---|---|
| `src/main.ts` | App bootstrap, `NestFactory.create`, `app.listen` | [01](01-project-setup-and-postgres-connection.md) |
| `package.json`, `tsconfig.json`, `nest-cli.json`, `.gitignore` | Project setup, TypeORM and `pg` dependencies, why no `.env` ships | [01](01-project-setup-and-postgres-connection.md) |
| `src/app.module.ts` | `ConfigModule.forRoot({ isGlobal: true })`, `TypeOrmModule.forRoot`, the missing `DATABASE_URL` | [01](01-project-setup-and-postgres-connection.md) |
| `src/app.controller.ts`, `src/app.service.ts` | The default, untouched Nest CLI starter controller and service | [01](01-project-setup-and-postgres-connection.md) |
| `slides/PostgreSQL Introduction.pdf` | What PostgreSQL is, why it is used, its SQL plus NoSQL hybrid features, what TypeORM is | [02](02-postgresql-and-relational-concepts-from-slides.md) |
| `src/employees/employees.entity.ts` | `@Entity`, `@Column`, `@PrimaryGeneratedColumn`, compared to Mongoose `@Schema`/`@Prop` | [03](03-typeorm-entities-vs-mongoose-schemas.md) |
| `src/user/user.entity.ts` | The same entity pattern, scaled down to two columns | [03](03-typeorm-entities-vs-mongoose-schemas.md), [06](06-user-entity-and-module-gap.md) |
| `src/employees/employees.module.ts` | `TypeOrmModule.forFeature`, how it creates an injectable `Repository<Employee>` | [04](04-employees-module-repository-and-service.md) |
| `src/employees/employees.service.ts` | `@InjectRepository`, full CRUD, `createQueryBuilder`, and two real string interpolation bugs | [04](04-employees-module-repository-and-service.md) |
| `src/employees/employees.controller.ts` | Every route, the one guarded route, and the missing `ParseIntPipe` on `:id` | [05](05-employees-controller-routes.md) |
| `src/user/user.module.ts` | Registers an entity with no service, controller, or exports, a working but unused stub | [06](06-user-entity-and-module-gap.md) |
| `src/auth/supabase-auth/supabase-auth.guard.ts` | `CanActivate`, bearer token parsing, local JWT verification with `jsonwebtoken` | [07](07-supabase-auth-guard.md) |
| Every `*.service.spec.ts` and `*.controller.spec.ts` under `src/employees` | Scaffold tests that fail once real constructor dependencies exist | [08](08-testing-gaps.md) |
| `src/auth/supabase-auth/supabase-auth.guard.spec.ts` | A spec that fails to even compile, not just fails at runtime | [08](08-testing-gaps.md) |
| `test/app.e2e-spec.ts` | End to end test, and why it needs a real `DATABASE_URL` to pass | [08](08-testing-gaps.md) |

## The single idea underneath this whole repo

If you take one thing away from this folder, it should be this: this project is a small, honest snapshot of a NestJS app in the middle of being built, not a finished, polished reference implementation. The `employees` feature shows the full, correct shape of a real TypeORM backed module end to end, entity, module, repository injected service, and controller, and it is worth learning that shape from it directly. But sitting right next to that complete example are exactly the kinds of small, believable mistakes a real project accumulates, a `user` module that only half exists, an auth guard wired onto a single route rather than the resource it seems meant to protect, two typoed error messages a few lines apart in the same service method, an unvalidated numeric route parameter, and test files that were generated once and never touched again after real dependencies were added. Learning to spot exactly those things, precisely, by reading the code rather than assuming it works because it looks complete, is as much a part of learning backend development as learning what `@Entity()` or `@InjectRepository()` do in the first place.
