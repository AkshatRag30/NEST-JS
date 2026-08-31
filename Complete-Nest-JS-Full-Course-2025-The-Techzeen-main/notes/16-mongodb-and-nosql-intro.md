# 16. MongoDB and NoSQL Introduction

## An important honesty note before anything else

Every other note in this folder ties a concept directly to real, working code inside `src`. This one is different: as of this snapshot of the repo, there is no actual MongoDB or Mongoose code anywhere in `src`. The `slides/MongoDB Introduction.pdf` deck covers the concept, and this project already stores everything (students, customers, products, categories) in plain in-memory arrays sitting directly inside services, for example `private students = [...]` in `src/student/student.service.ts`, or the connection status simulated with a boolean flag in `src/database/database.service.ts` (see [14-lifecycle-events.md](14-lifecycle-events.md)). This note explains the concept from the slides honestly, and then explains exactly what would need to be added to this project to make it real, without pretending code exists that does not. If a later version of this course repo adds real MongoDB integration, this is the note to come back and update with the real file paths.

## What "NoSQL" means

Most people's first exposure to databases is a relational (SQL) database, like MySQL or PostgreSQL, where data lives in rigid tables with predefined columns, and relationships between tables are explicit. NoSQL, from the slide deck, means "Not Only SQL." It refers to a whole family of databases that do not require that rigid table structure. Instead, NoSQL databases commonly store data in flexible formats, frequently JSON-like documents, and they do not require you to define every field up front with a fixed schema before you can store data.

## What MongoDB specifically is

MongoDB is the most widely used NoSQL database, and it is document based rather than table based. Instead of rows in a table, MongoDB stores JSON-like objects called documents, grouped into collections (the rough MongoDB equivalent of a table). A single "customer" document in MongoDB might look almost identical to the plain object this project already builds by hand in `CustomerService`:

```json
{
  "_id": "651f2b9e...",
  "name": "Bilal",
  "age": 30
}
```

Compare that to the interface already defined in this repo, `src/customer/interfaces/customer.interface.ts`:

```ts
export interface Customer{
    id: number;
    name: string;
    age: number;
}
```

The shape is almost the same conceptually, MongoDB's `_id` plays the same role this project's manually generated `id: Date.now()` plays in `CustomerService.addCustomer`, a unique identifier for each record. The core difference is that MongoDB is a real, persistent database. Right now, if you restart this app's server process, every customer ever added through `POST /customer` disappears instantly, because `private customers: Customer[] = []` is just a JavaScript array living in memory, and memory is wiped clean every time the process restarts. A real MongoDB collection would keep that same data safely on disk, across restarts, deployments, and even across multiple copies of your app running at once.

## Why MongoDB pairs especially well with NestJS

The slide deck's "Benefits with NestJS" points are genuinely accurate reasons this pairing is common in the Node.js ecosystem specifically:

MongoDB documents are already JSON-like, and JavaScript/TypeScript objects are also JSON-like, so there is very little translation needed between "the shape of my data in the database" and "the shape of my data in my code," unlike a SQL database where you often need a heavier mapping layer.

There is an official, well maintained integration package, `@nestjs/mongoose`, that wraps the popular `mongoose` library (a MongoDB tool for Node.js) in NestJS's module and dependency injection conventions, so connecting to MongoDB slots into the exact same `imports: [...]` pattern you already learned in [04-modules.md](04-modules.md) for `ConfigModule`.

MongoDB's schema can be changed without formal database migrations (the multi-step process SQL databases typically require to safely change a table's structure), which makes it faster to iterate on while a project's data model is still evolving, which is exactly the situation you are in as a learner building small features quickly.

## What it would actually take to add real MongoDB support to this project (illustrative only, not present in this repo)

This is a sketch of the shape the code would take, based on standard `@nestjs/mongoose` conventions, so you know what to look for or attempt yourself as a next step. None of the following file paths currently exist in this repo.

You would install `@nestjs/mongoose` and `mongoose`, then import `MongooseModule.forRoot(process.env.DATABASE_URL)` into `AppModule`, following the exact same `imports: [...]` pattern already used for `ConfigModule.forRoot({ isGlobal: true })`. This is precisely why `src/ev/ev.service.ts` already reads a `DATABASE_URL` environment variable through `ConfigService`, that value is meant to be a MongoDB connection string, ready for exactly this next step (see [15-environment-variables-and-config.md](15-environment-variables-and-config.md)).

Inside a feature module, like `CustomerModule`, you would define a schema class (using `@Schema()` and `@Prop()` decorators from `@nestjs/mongoose`) describing the shape of a document, register it with `MongooseModule.forFeature([...])` in that module's `imports`, and then inject a `Model<Customer>` into `CustomerService`'s constructor, the exact same constructor injection pattern from [07-dependency-injection.md](07-dependency-injection.md), just injecting a database model instead of a hand-written service.

`CustomerService`'s methods would then call real, asynchronous Mongoose model methods, `this.customerModel.find()` instead of `return this.customers`, and `this.customerModel.create(createCustomerDto)` instead of manually pushing onto an in-memory array, meaning those methods would need to become `async` and their return types would become `Promise<Customer[]>` and `Promise<Customer>`.

The DTO (`CreateCustomerDto`) and its `class-validator` decorators would not need to change at all, since validating incoming data (covered in [08-dto-and-interfaces.md](08-dto-and-interfaces.md) and [10-validation-and-pipes.md](10-validation-and-pipes.md)) is a concern completely separate from how that validated data eventually gets stored.

## Key terms recap

NoSQL: a category of databases that do not require the rigid tables and fixed schemas of traditional SQL databases.

Document: MongoDB's basic unit of storage, a JSON-like object.

Collection: MongoDB's rough equivalent of a table, a named group of documents.

Mongoose: the most common Node.js library for working with MongoDB, providing schemas and models on top of the raw database driver.

`@nestjs/mongoose`: the official NestJS package that wraps Mongoose in NestJS's module and dependency injection system.
