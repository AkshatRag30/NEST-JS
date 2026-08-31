# 14. Lifecycle Events

## What a lifecycle hook is

A NestJS application does not just spring into existence fully formed and then run forever unchanged. It goes through distinct stages: modules and their providers get created and initialized, the app finishes bootstrapping and starts accepting requests, and eventually the app shuts down. A lifecycle hook is a specially named method that NestJS will automatically call for you at one of these specific stages, if your class implements the matching interface. The slide deck's summary is exactly right: these are "special methods/hooks provided by NestJS, automatically called at different stages of a module/service/component's life, used to perform actions during creation or destruction."

## The lifecycle hooks the slides list, and what triggers each one

`onModuleInit()`, called once this module (and its dependencies) have been fully initialized.

`onModuleDestroy()`, called just before this module is torn down.

`onApplicationBootstrap()`, called once the entire application, all modules included, has finished initializing and is fully ready.

`onApplicationShutdown(signal?: string)`, called when the application is shutting down, and it receives the actual OS signal that triggered the shutdown (for example `'SIGINT'`, sent when you press Ctrl+C in the terminal).

Each of these corresponds to an interface you can implement on any provider: `OnModuleInit`, `OnModuleDestroy`, `OnApplicationBootstrap`, `OnApplicationShutdown`. Implementing the interface (which, structurally, just means having a method with the matching name) is what tells NestJS to actually call that method at the right time.

## Why you would ever need this: the `DatabaseService` example

This is the clearest real use case, and this repo has a working example of exactly this pattern, `src/database/database.service.ts`:

```ts
import { Injectable, OnModuleInit, OnApplicationShutdown } from '@nestjs/common';

@Injectable()
export class DatabaseService {
    private isConnected = false;

    onModuleInit(){
        this.isConnected = true;
        console.log('Database Connected!');
    }

    onApplicationShutdown(signal: string){
        this.isConnected = false;
        console.log(`Database Disconnected duw to app shutdown. Signal ${signal}`)
    }

    getStatus(){
        return this.isConnected ? 'Connected' : 'Disconnect';
    }
}
```

`implements OnModuleInit, OnApplicationShutdown` is not actually written here (notice the class does not literally say `implements OnModuleInit`), but the method names, `onModuleInit` and `onApplicationShutdown`, still match exactly what NestJS looks for, and NestJS calls lifecycle hooks based on whether a matching method exists on the class, not strictly on whether the `implements` keyword was written. Writing `implements OnModuleInit` is still considered good practice because it gets TypeScript to check that your method signature is correct, but functionally, having the correctly named method is what actually matters at runtime.

This service simulates a database connection with a simple boolean flag, `isConnected`, since this teaching project does not wire up a real database (see [16-mongodb-and-nosql-intro.md](16-mongodb-and-nosql-intro.md)). `onModuleInit()` runs the moment this module has finished initializing, flipping `isConnected` to `true` and logging `Database Connected!` to the console, exactly the point in a real app where you would actually open a real database connection. `onApplicationShutdown(signal)` runs when the whole application is shutting down, flipping `isConnected` back to `false` and logging the shutdown along with which signal caused it, exactly the point in a real app where you would cleanly close that real database connection so you do not leak resources or leave a connection hanging when the process exits.

`getStatus()` is a normal method with no lifecycle meaning at all, it simply reports the current value of `isConnected`, and it is exposed through `src/database/database.controller.ts`:

```ts
@Controller('database')
export class DatabaseController {
    constructor(private readonly databaseService: DatabaseService){}

    @Get('status')
    getStatus(){
        return {
            status: this.databaseService.getStatus(),
        }
    }
}
```

If you started this app and immediately requested `GET /database/status`, you would see `{"status": "Connected"}`, proving `onModuleInit` already ran before your request even had a chance to arrive, since module initialization always happens during startup, before the server starts accepting any traffic at all.

## The missing piece that makes shutdown hooks actually fire: `enableShutdownHooks()`

Go back to `main.ts`:

```ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  ...
  await app.listen(process.env.PORT ?? 3000);
  app.enableShutdownHooks();
}
```

This line is easy to miss, but it is required for `onApplicationShutdown` to ever be called at all. By default, NestJS does not automatically listen for OS-level termination signals. `app.enableShutdownHooks()` tells NestJS to start listening for signals like `SIGINT` (Ctrl+C) and `SIGTERM` (the signal most hosting platforms send when stopping a container), and when one arrives, to run every registered `onApplicationShutdown` hook across the whole app before the process actually exits. If this line were removed from `main.ts`, `DatabaseService`'s `onApplicationShutdown` would simply never run, no matter how the process was stopped, and you would lose the chance to do any clean shutdown work (closing database connections, finishing in-flight logs, and so on).

## When you would reach for each hook, practically

`onModuleInit`: the most commonly used hook. Good for anything that must be ready before the module starts actually handling requests, opening a database connection, warming a cache, loading configuration that requires an asynchronous call.

`onApplicationBootstrap`: less common, useful when you specifically need to know that every single module in the whole app (not just this one) has already finished its own initialization, for example if your logic depends on another, unrelated module having already set something up.

`onModuleDestroy` / `onApplicationShutdown`: for graceful cleanup, closing connections, flushing buffered logs, letting in-flight requests finish, so that stopping your app does not corrupt data or lose work silently.

## Key terms recap

Lifecycle hook: a specially named method NestJS automatically calls at a specific stage of a module's or the application's life.

`onModuleInit`: runs once a module (and what it depends on) has finished initializing.

`onApplicationShutdown`: runs when the application is shutting down, receiving the OS signal that triggered it.

`enableShutdownHooks()`: the call in `main.ts` required for shutdown-related lifecycle hooks to actually be triggered at all.
