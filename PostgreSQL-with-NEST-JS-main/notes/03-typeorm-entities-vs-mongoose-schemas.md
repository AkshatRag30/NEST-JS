# 03. TypeORM Entities, and How They Compare to Mongoose Schemas

This repo defines exactly two entities, `Employee` and `User`. Both use the same three decorators, `@Entity()`, `@PrimaryGeneratedColumn()`, and `@Column()`, all imported from `typeorm` rather than `@nestjs/typeorm`, which is worth noticing right away, the entity classes themselves are pure TypeORM, with no NestJS specific import anywhere in either file.

## `employees.entity.ts`, the full file

```ts
import { Entity, PrimaryGeneratedColumn, Column } from "typeorm";

@Entity()
export class Employee {
    @PrimaryGeneratedColumn()
    id: number;

    @Column()
    name: string;

    @Column()
    position: string;

    @Column()
    department: string;
}
```

`@Entity()` is a class decorator that tells TypeORM this class represents an actual database table. With no argument given, TypeORM derives the table name from the class name itself (lowercased, so this becomes an `employee` table). This is the direct equivalent of `@Schema()` in the MongoDB sibling project, both mark a plain class as "this describes one persisted record," but where `@Schema()` describes one MongoDB document, `@Entity()` describes one SQL table and, by extension, every row inside it.

`@PrimaryGeneratedColumn()` on `id` marks this column as the table's primary key, and tells TypeORM to let the database generate its value automatically, incrementing by one for every new row (under the hood, on Postgres, this becomes a `SERIAL` or `IDENTITY` integer column). This has no direct equivalent in the MongoDB sibling project's schemas, Mongoose never needs a decorator for this because every MongoDB document automatically gets an `_id` field of type `ObjectId` whether you ask for it or not. TypeORM entities, by contrast, need you to explicitly declare which column is the primary key and how its value gets generated, nothing is automatic unless you decorate it.

`@Column()` on `name`, `position`, and `department`, each with no options passed in, marks each property as an ordinary stored column. TypeORM infers the underlying SQL column type from the TypeScript type of the property itself, since all three are typed `string`, they become `varchar` columns in Postgres. This is the direct equivalent of `@Prop()` in the sibling project's Mongoose schemas, both mark "this field actually gets persisted." One real difference worth noticing: nowhere in `@Column()` here is `{ nullable: false }` or any equivalent of Mongoose's `{ required: true }` written explicitly. By default, a plain `@Column()` in TypeORM is `NOT NULL`, so `name`, `position`, and `department` are implicitly required at the database level even without writing that requirement out, whereas the MongoDB sibling project's `Student` schema had to spell out `@Prop({ required: true })` on every field it wanted enforced, TypeORM's default leans the opposite direction, non nullable unless you explicitly add `{ nullable: true }`.

## `user.entity.ts`, the full file

```ts
import { Entity, PrimaryGeneratedColumn, Column } from "typeorm";

@Entity()
export class User{
    @PrimaryGeneratedColumn()
    id: number;

    @Column()
    name: string
}
```

This is the same exact pattern, scaled down to just two columns, `id` and `name`. There is genuinely nothing more to this entity, no email, no password, no relationship to `Employee`, it is a minimal placeholder shape. What actually happens (and does not happen) with this entity once it exists is covered fully in [06-user-entity-and-module-gap.md](06-user-entity-and-module-gap.md).

## What neither entity uses

Neither `Employee` nor `User` uses `@CreateDateColumn()` or `@UpdateDateColumn()`, TypeORM's equivalent of Mongoose's `@Schema({ timestamps: true })` option from the sibling project's `Student` schema, which automatically stamped every document with `createdAt` and `updatedAt`. Neither entity here tracks when a row was created or last changed at all. Neither entity uses any TypeORM relationship decorator either, `@OneToOne()`, `@OneToMany()`, `@ManyToOne()`, or `@ManyToMany()`, the tools that would let one entity reference rows in another table, the way the sibling MongoDB project's `employee`/`profile` and `project`/`developer` modules demonstrated one to one and many to many referencing. `Employee` and `User` in this project are two completely independent tables with no relationship to each other defined anywhere in the code.

## The side by side comparison, for a reader who already knows the Mongoose version

| Concept | Mongoose (`@nestjs/mongoose`) | TypeORM (`@nestjs/typeorm`) |
|---|---|---|
| Mark a class as a persisted unit | `@Schema()` | `@Entity()` |
| Mark a field as stored | `@Prop()` | `@Column()` |
| Unit of storage | one flexible document | one row in a fixed table |
| Unique identifier | automatic `_id: ObjectId` on every document | explicit `@PrimaryGeneratedColumn()`, you choose the column and its generation strategy |
| Make a field required | `@Prop({ required: true })` | plain `@Column()` is already `NOT NULL` by default |
| Automatic timestamps | `@Schema({ timestamps: true })` | `@CreateDateColumn()` / `@UpdateDateColumn()`, not used anywhere in this repo |
| Compile the class into something usable | `SchemaFactory.createForClass(Student)` | nothing extra needed, the decorated class is handed directly to TypeORM |
| Relating one unit to another | an embedded subdocument, or a `Types.ObjectId` field with `ref:` | a relationship decorator such as `@OneToMany()`, not used anywhere in this repo |

That last row is the biggest thing to take away if you are coming from the MongoDB project: this particular repo never actually demonstrates a TypeORM relationship. `Employee` and `User` are two flat, unrelated tables. If you want to see how TypeORM expresses a real relationship the way the sibling project's `employee`/`profile` or `project`/`developer` modules did for Mongoose, that would be the natural next thing to look for in a more advanced TypeORM example, it simply is not present in this codebase as it stands.
