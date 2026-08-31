# 06. Embedding Subdocuments

Starting with this note, the remaining topics move from "how do you talk to MongoDB at all" into the actual subject line of this whole repo: how do you model relationships between pieces of data in MongoDB. The first strategy, embedding, means storing related data directly inside the same parent document rather than as a separate document in its own collection. This repo demonstrates two flavors of embedding: a single embedded object (`user`/`address`) and an array of embedded objects (`product`/`tag`).

## A single embedded subdocument: `User` and `Address`

```ts
// src/user/schemas/address.schema.ts
@Schema()
export class Address {
    @Prop()
    street: string;

    @Prop()
    city: string;
}
```

```ts
// src/user/schemas/user.schema.ts
@Schema()
export class User extends Document {
    @Prop()
    name: string;

    @Prop({ type: Address })
    address: Address
}

export const UserSchema = SchemaFactory.createForClass(User);
```

`Address` is decorated with `@Schema()` and `@Prop()` just like any other schema class (see [03-schemas-and-props.md](03-schemas-and-props.md)), but notice it has no `SchemaFactory.createForClass()` call of its own and is never registered with `MongooseModule.forFeature()` anywhere (check `src/user/user.module.ts`, only `User` is registered there). That absence is exactly what makes `Address` an embedded subdocument rather than a referenced one: it never gets its own MongoDB collection, it only ever exists nested inside a `User` document.

The line that actually wires the two together is `@Prop({ type: Address })` on `User.address`. This tells Mongoose that the `address` field is not a simple scalar value (a string, number, and so on), it is an entire nested object shaped like the `Address` class, complete with its own `street` and `city` fields and their own validation rules, all stored inside the same JSON-like document as the user's `name`.

Here is what a saved `User` document actually looks like inside MongoDB as a result:

```json
{
  "_id": "...",
  "name": "Farzeen Ali",
  "address": {
    "street": "123 Street",
    "city": "Karachi"
  },
  "__v": 0
}
```

Everything lives inside one single document. Look at how `user.service.ts` creates one:

```ts
async createUser(): Promise<User>{
    const user = new this.userModel({
        name: 'Farzeen Ali',
        address: {
            street: '123 Street',
            city: 'Karachi'
        }
    })
    return user.save();
}
```

The `address` key is just a plain nested object literal passed straight into the model constructor, exactly like any other field, and one single `.save()` call persists the user and its address together, in one write, to one document. There is no second query, no linking step, no `populate()` call anywhere in this module (compare that to the referencing modules in the next two notes, where `populate()` shows up constantly). `findAll` similarly does nothing special, `this.userModel.find()` returns every user with its address already fully present, because the address was never separated out in the first place.

## An embedded array of subdocuments: `Product` and `Tag`

```ts
// src/product/schemas/tag.schema.ts
@Schema()
export class Tag {
    @Prop()
    name: string;
}
```

```ts
// src/product/schemas/product.schema.ts
@Schema()
export class Product extends Document{
    @Prop()
    title: string;

    @Prop({ type: [Tag]})
    tags: Tag[];
}
```

This is the same embedding idea as `Address`, but for a one to many relationship instead of one to one: a single product can have many tags. The only real difference in the code is `@Prop({ type: [Tag] })` instead of `@Prop({ type: Address })`, wrapping `Tag` in square brackets tells Mongoose this field is an array of embedded `Tag` subdocuments rather than a single one.

```ts
// src/product/product.service.ts
async createProduct(): Promise<Product> {
    const product = new this.productModel({
        title: 'Gaming Laptop',
        tags: [
            { name: 'electronics'},
            { name: 'gaming'},
            { name: 'laptop'},
        ]
    })
    return product.save();
}
```

Again, one call to `.save()` persists the product and all three of its tags together in a single document, no linking step needed, and `getAllProducts`'s plain `this.productModel.find()` gets every product back with its tags already attached, exactly as embedding is supposed to work.

## Why you would choose embedding, and its real tradeoff

Both examples share the same shape of decision: `Address` only ever makes sense in the context of a specific `User`, and a `Tag`'s `name` field only ever gets read as part of looking at a `Product`; nothing in this repo ever needs to query "give me every address in the system" or "give me every tag in the system" independent of their parent. That is exactly the situation embedding is good for: related data that is always read together and is not updated independently of its parent, which is precisely the "Pros of Embedding" and "When to Use Embedding" guidance from the lecture slides, covered in full with the tradeoffs against referencing in [10-relationship-concepts-from-lecture.md](10-relationship-concepts-from-lecture.md). The cost of this choice, also called out in those slides, is that if `Address` or `Tag` ever needed to grow into something with many more fields, or something referenced by multiple unrelated documents, embedding would start working against you, at which point the referencing patterns from the next two notes become the better tool.
