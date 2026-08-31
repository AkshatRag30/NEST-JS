# 04. PrismaService and PrismaModule

Prisma's generated client is a plain TypeScript class, `PrismaClient`, with no knowledge of NestJS at all on its own. This note covers the two small files this project uses to turn that plain class into a proper, injectable piece of the Nest dependency injection system, `src/prisma/prisma.service.ts` and `src/prisma/prisma.module.ts`.

## `prisma.service.ts`, the whole file

```ts
import { Injectable, OnModuleDestroy, OnModuleInit } from '@nestjs/common';
import { PrismaClient } from 'generated/prisma';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy{
    async onModuleInit() {
        await this.$connect();
    }
    async onModuleDestroy() {
        await this.$disconnect();
    }
}
```

`import { PrismaClient } from 'generated/prisma';` is pulling in the generated client class from the custom output folder configured in `schema.prisma`'s generator block, covered in the previous note, rather than from the `@prisma/client` package name most Prisma tutorials use.

`export class PrismaService extends PrismaClient` is the core of the whole pattern: `PrismaService` does not hold a `PrismaClient` instance as a private field somewhere inside it, it directly extends `PrismaClient`, meaning a `PrismaService` instance genuinely is a `PrismaClient`, with every single generated method Prisma produced from `schema.prisma`, `this.book.findMany()`, `this.book.create()`, and so on, available directly on `this` inside `PrismaService`, and by extension on any injected `PrismaService` instance anywhere else in the app, exactly what `book.service.ts` relies on in the next note.

`implements OnModuleInit, OnModuleDestroy` connects this class to two of the lifecycle hook interfaces covered in the Techzeen sibling project's lifecycle notes. Writing `implements` explicitly here, rather than just relying on the method names matching as that project's own `DatabaseService` example did, gets TypeScript to check that `onModuleInit` and `onModuleDestroy` actually have the signatures Nest expects.

`onModuleInit()` runs `await this.$connect()`. `$connect` is one of the handful of special methods every generated Prisma Client exposes (prefixed with `$` to keep them visually distinct from your own model methods like `book`), and it is the call that actually opens a real connection pool to the Postgres database at whatever URL `DATABASE_URL` resolves to. Because `onModuleInit` fires automatically once this module has finished initializing, which happens during `NestFactory.create(AppModule)` at application startup, this means the database connection is opened exactly once, automatically, before the app ever starts accepting HTTP or GraphQL requests, with no feature service anywhere needing to remember to connect anything itself. This is also, concretely, the exact point where the missing `DATABASE_URL` gap from the previous note actually becomes a real, live problem: `$connect()` is precisely the call that throws the `Environment variable not found: DATABASE_URL` error reproduced in that note, meaning this app, as shipped, cannot finish starting up at all without a working connection string.

`onModuleDestroy()` runs `await this.$disconnect()`, cleanly closing that same connection pool when this module is torn down. As noted in [01-project-setup-and-bootstrap.md](01-project-setup-and-bootstrap.md), `main.ts` never calls `app.enableShutdownHooks()`, so in practice this specific method would never actually get a chance to run when the process receives a termination signal like Ctrl+C, it is only guaranteed to run in more controlled scenarios, such as a NestJS testing module being explicitly closed.

## Why wrap `PrismaClient` in an injectable service at all

This pattern, extending `PrismaClient` inside a small `@Injectable()` class with `OnModuleInit`/`OnModuleDestroy` wired up, is Prisma's own officially recommended way to use it inside a NestJS application, and the reasoning is straightforward once you have already seen the MongoDB sibling project's `@InjectModel()` pattern: a raw `new PrismaClient()` created somewhere ad hoc, with nobody managing when it connects or disconnects, would have no natural hook into Nest's own startup and shutdown sequence, and every feature module that wanted to use it would need to construct or share its own instance manually. By instead registering `PrismaService` as a real Nest provider, the dependency injection container becomes responsible for creating exactly one shared instance, injecting that same instance into any constructor that asks for it by type (`private prisma: PrismaService`, seen directly in `book.service.ts`), and automatically running `$connect()` once at startup and `$disconnect()` once at shutdown. No feature service anywhere in this codebase has to think about connection management at all, it just receives an already connected `PrismaService` and starts calling query methods on it.

## `prisma.module.ts`, and whether it is global

```ts
import { Module } from '@nestjs/common';
import { PrismaService } from './prisma.service';

@Module({
  providers: [PrismaService],
  exports: [PrismaService]
})
export class PrismaModule {}
```

Reading this file directly, rather than assuming, there is no `@Global()` decorator anywhere above `@Module(...)`, and no import of `Global` from `@nestjs/common` at the top of the file at all. `PrismaModule` is an ordinary, scoped Nest module. It registers `PrismaService` as a provider and `exports` it so that other modules can use it, but because it is not global, `PrismaService` is only actually injectable inside a module that explicitly lists `PrismaModule` in its own `imports` array.

This works correctly in this specific repository only because `book.module.ts` does exactly that (covered in the next note), it imports `PrismaModule` directly. But it is worth being explicit that this is a real, observable detail rather than an assumption of best practice: many teaching examples and Prisma's own NestJS recipe specifically recommend marking a database service module like this one as `@Global()`, precisely because a database connection is exactly the kind of thing you want available everywhere in an application without repeating the same import in every single feature module you ever add. As this project currently stands, with only one feature module, that cost is not visible, but any second or third feature module added later to this codebase would have to remember, on its own, to add `PrismaModule` to its own `imports` array, there is nothing automatic enforcing that the way a global module would provide.
