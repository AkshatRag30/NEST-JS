# 1. Introduction to NestJS

## What problem are we even solving

Before touching NestJS, it helps to know what it is replacing. Node.js by itself is just a JavaScript runtime, it does not know anything about HTTP servers, routes, or requests. Most people's first backend framework is Express, which is very minimal. In Express, you write something like this:

```js
app.get('/products', (req, res) => {
  res.send('list of products');
});
```

That is fine for a small app, but once your app grows to dozens of routes, services, database calls, and validation rules, a plain Express app turns into a pile of loosely organized functions in a few giant files. There is no enforced structure, no built in way to separate "business logic" from "routing logic", and no standard pattern for wiring pieces together. Every developer on a team ends up organizing things differently.

## What NestJS is

NestJS is a progressive Node.js framework for building efficient and scalable server side applications. It is built with TypeScript (so you get type safety) and it borrows its architectural style heavily from Angular, the frontend framework. That is why NestJS uses the same ideas as Angular: decorators, modules, and dependency injection.

Concretely, NestJS is a layer on top of Node.js (and, underneath, on top of Express by default, see `@nestjs/platform-express` in `package.json`) that gives you:

1. A required project structure, so every NestJS app looks similar and any NestJS developer can jump into any NestJS codebase and immediately understand where things live.
2. A dependency injection system, so classes do not need to manually create the other classes they depend on.
3. Decorators, small annotations like `@Controller()` or `@Get()` that tell NestJS how to treat a class or a method, without you writing manual wiring code.
4. First class TypeScript support, catching mistakes at compile time instead of at runtime.

## Why we need NestJS specifically

From the slides in `slides/Nest JS Introduction.pdf`, the pitch for NestJS boils down to three things:

1. It simplifies backend development by giving you modern, opinionated architecture instead of a blank page.
2. It gives you a structured way to build applications that are both scalable (can grow without turning into spaghetti) and testable (each piece can be tested in isolation).
3. It solves the organizational limitations of a raw Express app by forcing you to split code into modules, controllers, and services from day one.

## Benefits, translated into plain English

The slide deck lists these benefits: full TypeScript support, a built in dependency injection system, easy integration with databases/WebSockets/GraphQL/microservices, a scalable and maintainable codebase, and an active community. In practice, here is what that means for you as a learner:

Full TypeScript support means the editor will warn you the moment you pass the wrong type into a function, before you even run the code. Look at `src/customer/interfaces/customer.interface.ts` in this repo: it defines exactly what shape a `Customer` object must have (`id`, `name`, `age`). If you tried to return an object missing `age`, TypeScript would refuse to compile.

Built in dependency injection means when `CustomerController` needs to talk to `CustomerService`, you never write `new CustomerService()` yourself. NestJS creates it and hands it to the controller automatically. This is covered in full detail in [07-dependency-injection.md](07-dependency-injection.md).

Easy integration with databases and other systems means there are official packages (like `@nestjs/config` used in this repo, see `src/ev/ev.service.ts`) that plug straight into NestJS's module system, instead of you gluing third party libraries together by hand.

## How this maps to the code in this repo

This project is a teaching repo, so it deliberately contains many small, independent examples rather than one large production app. As you go through these notes you will see the same shape repeated over and over: a `*.module.ts` file, a `*.controller.ts` file, and a `*.service.ts` file, for features like `category`, `customer`, `employee`, `student`, `product`, and more. That repetition is intentional. Once the pattern clicks for one feature, it clicks for all of them, because NestJS is fundamentally the same handful of building blocks arranged over and over.

The very first file to actually run when you start this project is `src/main.ts`. That is where the next note picks up.

## Key terms recap

Framework: a set of reusable code and conventions you build your application inside of, instead of writing everything from scratch.

Progressive framework: NestJS's own description of itself, meaning it does not force one specific way of doing everything (you can still drop down to raw Express style code if you need to), it just gives strong defaults.

Decorator: a special piece of syntax starting with `@` that attaches metadata or behavior to a class, method, or property. Explained in depth once we hit controllers.

TypeScript: a superset of JavaScript that adds static types, meaning types are checked while you are writing code, not only while the program is running.
