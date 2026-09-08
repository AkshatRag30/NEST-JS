# Phase 1 Prompt: Foundations and Config

Use this in a Claude Code session opened at the `LearnBridge-Capstone` folder, which right now only contains a `planning` folder and nothing else. This prompt takes it from empty to a booting, empty shell application with every cross cutting concern wired in.

Copy everything inside the fenced block below into Claude Code as one message.

```
Before writing any code, read these files completely: planning/01-PRD-master.md, planning/02-tech-stack-and-architecture.md, and planning/05-phase-1-PRD-foundations-and-config.md. This is phase 1 of a multi phase build, do not build anything from later phases, no feature modules like users, auth, or courses yet, this phase only sets up the empty shell.

Scaffold a fresh NestJS project in this exact folder (LearnBridge-Capstone), using the standard Nest CLI conventions, package.json, tsconfig.json, nest-cli.json, src/main.ts, src/app.module.ts, src/app.controller.ts, src/app.service.ts, and a test folder with an e2e config, the same shape you would get from running the Nest CLI generator. If the folder already having a planning subdirectory prevents `nest new .` from running cleanly, scaffold the files by hand instead so the end result matches what that command would have produced.

Install these dependencies: @nestjs/config, @prisma/client, prisma as a dev dependency, @nestjs/mongoose, mongoose, class-validator, class-transformer. Use current stable versions compatible with NestJS 11.

Run prisma init to create a prisma/schema.prisma file with a postgresql datasource reading from env("DATABASE_URL") and a client generator, but do not add any models to it yet, that starts in phase 2.

Set up ConfigModule.forRoot in AppModule with isGlobal set to true, and a validate function or a Joi schema that checks for DATABASE_URL, MONGO_URI, and JWT_SECRET, throwing a clear error at startup naming exactly which variable is missing if any of them are absent, even though JWT_SECRET is not used by any code until phase 2, it must already be required now so the app never boots without it later. Create a .env.example file at the project root listing all three variable names with placeholder values and a one line comment explaining each, but do not create a real .env file, that stays local and untracked, add it to .gitignore if it is not already there.

Create src/prisma/prisma.service.ts as a class extending PrismaClient, implementing OnModuleInit, calling this.$connect() and logging one clear success line in onModuleInit. Create src/prisma/prisma.module.ts, decorated with @Global(), providing and exporting PrismaService, and import it once into AppModule.

Connect to MongoDB using MongooseModule.forRootAsync in AppModule, reading MONGO_URI through ConfigService inside the factory function, not through a direct process.env read anywhere.

Create a logging middleware, applied globally, that logs the HTTP method, path, status code, and response time for every request. Create a global exception filter implementing ExceptionFilter, catching both Nest's HttpException family and any unexpected error, and always responding with one consistent JSON shape containing statusCode, message, and a timestamp, applied in main.ts with app.useGlobalFilters. Register a global ValidationPipe in main.ts with whitelist true, forbidNonWhitelisted true, and transform true.

Write one unit test for the environment validation function or schema in isolation, asserting it throws when a required variable is missing and passes when all three are present, this does not require a real database connection to test.

When you are done, tell me in plain terms what you built, confirm the app fails to start with a clear named error when a required env var is missing, and list the exact npm scripts I should run to start it locally once I have created my own real .env file.
```
