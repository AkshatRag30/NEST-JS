# 10. Data Relationships in MongoDB, From the Lecture Slides

The `lectures` folder contains one slide deck, `Data Relationships in MongoDB.pdf`, from the same TechZeen course this repo's code is built around. It is short, thirteen slides, mostly bullet points, and it lays out the conceptual vocabulary for everything demonstrated in [06-embedding-subdocuments.md](06-embedding-subdocuments.md) through [08-many-to-many-referencing.md](08-many-to-many-referencing.md). This note walks through the deck's own structure and points each slide at the specific code that puts it into practice.

## What a "data relationship" means here

The deck opens by defining data relationships plainly, as "connections between documents in collections." This is the same idea you would call a relationship in a SQL database, two pieces of data that are meaningfully linked, customer and order, book and library, developer and project, but the slide's point is that MongoDB, being a document database rather than a table based one (see any general MongoDB introduction for that background), does not have a single, forced way to represent that link the way a SQL foreign key does. Instead, you choose between two fundamentally different strategies, and this repo's six feature modules exist specifically to show both strategies in action.

## Embedding

The deck defines embedding as "storing related data inside the same document." This is exactly what [06-embedding-subdocuments.md](06-embedding-subdocuments.md) covers with `User.address` (a single embedded `Address`) and `Product.tags` (an array of embedded `Tag`s). In both cases, one `.save()` call writes the parent and its related data together, in one document, in one write.

**Pros of Embedding**, per the deck: fast data retrieval, since everything comes back in a single document with a single query, no second lookup needed, and it is simple to reason about for small related data. You can see this directly in `user.service.ts`'s `findAll()`, a completely plain `this.userModel.find()` with no `.populate()` call anywhere, because there is nothing to populate, the address data was never separated from the user in the first place.

**Cons of Embedding**, per the deck: document size grows as more related data piles up, and it becomes hard to update deeply nested data. This repo's examples are deliberately small enough that neither problem actually bites (a `Product` with three tags is tiny), but you can reason about where the line would be crossed: if `Product.tags` needed to grow into hundreds of entries, or if `Tag` itself grew complex enough that you regularly needed to update one tag's data independently of the specific product it happened to be embedded in, embedding would start costing more than it saves.

**When to Use Embedding**, per the deck: when related data is mostly read together, and when that data is small and not updated frequently. Both `Address` and `Tag` fit this description exactly, an address is only ever meaningful in the context of its one user, and nothing in this repo ever needs to update a tag's name independently of the product that holds it.

## Referencing

The deck defines referencing as "storing related data in separate documents and linking by IDs." This is the strategy behind [07-one-to-one-and-one-to-many-referencing.md](07-one-to-one-and-one-to-many-referencing.md) (`Employee`/`Profile` and `Library`/`Book`) and [08-many-to-many-referencing.md](08-many-to-many-referencing.md) (`Project`/`Developer`). In every one of these examples, the related schema (`Profile`, `Book`, `Developer`) gets its own `SchemaFactory.createForClass()` call and its own entry in `MongooseModule.forFeature([...])`, meaning it lives in its own MongoDB collection, and the parent only stores a `Types.ObjectId` (or an array of them) with a `ref:` pointing at that collection.

**Pros of Referencing**, per the deck: it keeps documents small, since a `Library` document never has to physically contain every one of its books' full data, only their ids, and it is easier to update and manage large data, because updating a `Book`'s title only ever means writing to that one `Book` document, regardless of how many libraries happen to reference it.

**Cons of Referencing**, per the deck: it requires extra queries or joins, explicitly calling out `populate()` in Mongoose by name, exactly the method used in `employee.service.ts`'s `findAll()`, `library.service.ts`'s `getLibraries()`, and `project.service.ts`'s `getDevelopers()`/`getProjects()`. Each of those calls is a second query happening behind the scenes, resolving ObjectIds back into full documents, which the deck summarizes as "slightly slower reads" compared to embedding's single document fetch.

**When to Use Referencing**, per the deck: when related data is large or updated independently, and when you need to maintain data normalization (avoiding the same piece of data being duplicated in multiple places). `Book` fits this well, a book could in principle belong to more than one library's catalog, or be looked up and edited on its own regardless of which library holds it, and duplicating its full title and author into every library that references it would risk those copies drifting out of sync with each other, exactly the kind of duplication normalization is meant to avoid.

## The three relationship shapes, mapped onto this repo's modules

The deck names three relationship shapes without going deep into each, one to one, one to many, and many to many. This repo happens to provide one clean, real code example of each:

One to one: `Employee` and `Profile` in [07-one-to-one-and-one-to-many-referencing.md](07-one-to-one-and-one-to-many-referencing.md). Each employee has exactly one profile, stored as a single `ObjectId` field (`profile: Profile`, not an array).

One to many: `Library` and `Book`, also in [07-one-to-one-and-one-to-many-referencing.md](07-one-to-one-and-one-to-many-referencing.md). One library holds an array of book references, but nothing on `Book` points back at which library (or libraries) hold it, the reference only flows in one direction.

Many to many: `Project` and `Developer` in [08-many-to-many-referencing.md](08-many-to-many-referencing.md). Both sides hold an array of references to the other, a developer's `projects` array and a project's `developers` array, and as that note explains, keeping both arrays consistent with each other is work the application code has to do itself, which is exactly why `project.service.ts`'s `seed()` method needs three separate stages instead of one.

Worth noticing: the embedding examples (`User`/`Address`, `Product`/`Tag`) are also, technically, one to one and one to many relationships, just implemented with embedding instead of referencing. The deck's three relationship shapes and its two storage strategies (embedding versus referencing) are two independent axes, not the same choice, any of the three shapes can in principle be built either way, and this repo happens to demonstrate one to one and one to many built both ways (embedded for `User`/`Product`, referenced for `Employee`/`Library`), while many to many only appears built the referenced way, which makes sense, since embedding does not scale well to a relationship where both sides can have many partners on the other side, you would end up duplicating the same data in many places at once.
