# MongoDB with NestJS — Study Guide

This folder holds a complete set of topic by topic notes built from this repository. Unlike a plain intro course, this project is entirely about one thing: how NestJS talks to a real MongoDB database through the `@nestjs/mongoose` package, and how you model relationships between documents (embedding versus referencing, one to one, one to many, and many to many). There is a `lectures` folder with one PDF, `Data Relationships in MongoDB.pdf`, and six small feature modules under `src` (`student`, `user`, `employee`, `product`, `library`, `project`) that each demonstrate one specific Mongoose pattern.

Every note below explains a concept and then points at the exact file and lines in this repo where that concept is used, so you can open the code side by side with the note.

## How to read these notes

Go in order the first time. Each note builds on the one before it.

1. [01-project-setup-and-bootstrap.md](01-project-setup-and-bootstrap.md) — what this project is, `package.json`, `main.ts`, `tsconfig.json`, and the npm scripts.
2. [02-connecting-to-mongodb.md](02-connecting-to-mongodb.md) — `MongooseModule.forRoot()`, the `MONGO_URI` environment variable, and how `AppModule` wires everything together.
3. [03-schemas-and-props.md](03-schemas-and-props.md) — `@Schema()`, `@Prop()`, `SchemaFactory.createForClass()`, and the two different ways this repo types a Mongoose document.
4. [04-forfeature-and-dependency-injection.md](04-forfeature-and-dependency-injection.md) — `MongooseModule.forFeature()` and `@InjectModel()`, how a Mongoose model becomes an injectable NestJS provider.
5. [05-crud-with-mongoose-models.md](05-crud-with-mongoose-models.md) — the full create, read, update, delete flow, using the `student` module as the reference example, including the difference between `PUT` and `PATCH` in this codebase.
6. [06-embedding-subdocuments.md](06-embedding-subdocuments.md) — storing related data directly inside a parent document, using the `user`/`address` and `product`/`tag` modules.
7. [07-one-to-one-and-one-to-many-referencing.md](07-one-to-one-and-one-to-many-referencing.md) — storing related data in a separate collection and linking by ObjectId, using the `employee`/`profile` (one to one) and `library`/`book` (one to many) modules.
8. [08-many-to-many-referencing.md](08-many-to-many-referencing.md) — the `project`/`developer` module, where each side holds an array of references to the other.
9. [09-testing-gaps.md](09-testing-gaps.md) — an honest look at the `.spec.ts` files in this repo, including a real gap: several of these tests will fail the moment you actually run them.
10. [10-relationship-concepts-from-lecture.md](10-relationship-concepts-from-lecture.md) — the concepts from the `Data Relationships in MongoDB.pdf` slide deck, mapped onto the code you already read.
11. [11-full-recap-and-file-map.md](11-full-recap-and-file-map.md) — the life of one HTTP request from route to database and back, plus a table mapping every file in `src` to the concept it teaches.

## A note on the source material

This repo's `lectures` folder contains a single short slide deck (again from TechZeen), which is high level and mostly bullet points about embedding versus referencing. All of the real depth in these notes comes from reading the actual working code in `src`, which goes well beyond the slides by showing six concrete, runnable examples of different relationship shapes. Where the code does something a beginner might find surprising, or where the code has a genuine bug or gap, that is called out honestly rather than glossed over.
