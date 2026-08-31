# 06. The User Entity and Module, What Actually Exists

The `user` folder is the smallest thing in this repo, two files, `user.entity.ts` and `user.module.ts`. It is worth reading both completely, because what is missing here is just as important as what is present.

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

This is a minimal entity, exactly the same pattern explained in [03-typeorm-entities-vs-mongoose-schemas.md](03-typeorm-entities-vs-mongoose-schemas.md), `@Entity()` marking this as a real table (named `user`), `@PrimaryGeneratedColumn()` giving it an auto incrementing integer primary key, and one single `@Column()`, `name`. There is no email, no password, no relationship to `Employee`, nothing else.

## `user.module.ts`, the full file

```ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './user.entity';

@Module({
    imports: [ TypeOrmModule.forFeature([User])],
})
export class UserModule {}
```

This is the entire module, and it is worth being precise about exactly what it does and does not contain, since it would be easy to assume a module named `UserModule` must have a service and a controller behind it somewhere. It does not. There is no `providers` array, so there is no `UserService` anywhere in this codebase, none exists to write. There is no `controllers` array, so there are no HTTP routes for users at all, no `GET /user`, no `POST /user`, nothing, searching this entire repository confirms there is no `user.controller.ts` and no `user.service.ts` file anywhere. There is also no `exports` array, which matters just as much as the missing providers and controllers.

## What `TypeOrmModule.forFeature([User])` alone actually accomplishes

Registering `User` here does still do something real. Because `AppModule` sets `autoLoadEntities: true` on `TypeOrmModule.forRoot({...})` (see [01-project-setup-and-postgres-connection.md](01-project-setup-and-postgres-connection.md)), and because `AppModule` imports `UserModule`, the `User` entity genuinely does get picked up by the main database connection. Combined with `synchronize: true`, this means a `user` table really would get created in the connected Postgres database the moment this app starts successfully. So `UserModule` is not completely inert, it does have one real effect on the database schema.

What it does not accomplish is giving anything, anywhere, a way to actually interact with that table through this application. There is no service in this codebase that injects `Repository<User>` (compare this to `EmployeesService`'s `@InjectRepository(Employee)`, covered in [04-employees-module-repository-and-service.md](04-employees-module-repository-and-service.md), which is exactly the piece missing here). Even if some other module wanted to inject `Repository<User>` in the future, `UserModule` as written could not supply it to them, because `TypeOrmModule.forFeature([User])` is only listed in `imports`, and `UserModule` has no `exports: [TypeOrmModule]` line re-exposing that registration to anything that might import `UserModule` itself. As it stands, the `Repository<User>` provider that `forFeature` creates is usable only from directly inside `UserModule`, and since `UserModule` defines no provider of its own to actually use it, that repository provider currently has no consumer anywhere in this application.

## The honest summary

`UserModule` is best understood as a stub, or a placeholder, rather than a working feature. It demonstrates the first half of wiring up a new TypeORM entity into a NestJS app, defining the entity class and registering it with `forFeature()`, and it does cause a `user` table to actually exist in the database once the app connects successfully, but it stops there. There is no CRUD, no routes, no business logic, and nothing else in this codebase currently depends on it. If a beginner were extending this repo as an exercise, the natural next step would be exactly what `EmployeesModule` already shows in full, add a `UserService` with `@InjectRepository(User)`, add a `UserController` with real routes, and add both to `UserModule`'s `providers` and `controllers` arrays, the same shape the employees module already demonstrates end to end.
