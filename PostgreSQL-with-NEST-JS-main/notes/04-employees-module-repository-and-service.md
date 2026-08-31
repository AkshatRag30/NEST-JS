# 04. The Employees Module, the Repository Pattern, and the Service

The `employees` folder is the one fully worked out feature in this repo, an entity, a module, a service, and a controller all wired together the way a real NestJS feature is meant to look. This note covers the module wiring and the service, the next note covers the controller's actual routes.

## `employees.module.ts`

```ts
import { Module } from '@nestjs/common';
import { EmployeesService } from './employees.service';
import { EmployeesController } from './employees.controller';
import { TypeOrmModule } from '@nestjs/typeorm';
import { Employee } from './employees.entity';

@Module({
  imports: [TypeOrmModule.forFeature([Employee])],
  providers: [EmployeesService],
  controllers: [EmployeesController]
})
export class EmployeesModule {}
```

`TypeOrmModule.forFeature([Employee])` is the direct TypeORM equivalent of `MongooseModule.forFeature([...])` from the sibling project. Where `MongooseModule.forRoot()` establishes the one shared connection for the whole app, `forFeature()` is what you call inside a specific feature module to say "within this module, I want to work with rows shaped like this entity." Passing `[Employee]` here (just the class itself, not an object with `name`/`schema` keys the way Mongoose needed) is what causes NestJS's dependency injection container to create and register an injectable `Repository<Employee>` provider, scoped to this module, under an internal token derived from the `Employee` class. That is exactly the provider `@InjectRepository(Employee)` asks for by name inside `employees.service.ts` below.

`providers: [EmployeesService]` and `controllers: [EmployeesController]` are the ordinary NestJS module wiring you would expect from any feature module, nothing TypeORM specific about either line.

## The repository pattern and `@InjectRepository()`

```ts
import { Injectable, NotFoundException } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Employee } from './employees.entity';
import { Repository } from 'typeorm';

@Injectable()
export class EmployeesService {
    constructor(
        @InjectRepository(Employee)
        private employeeRepository: Repository<Employee>
    ) {}
    // ...
}
```

`Repository<Employee>` is TypeORM's own class representing exactly what the name suggests, a repository, an object that knows how to find, create, update, and delete rows of one specific entity type, `Employee` in this case. This is the same architectural idea as `Model<StudentDocument>` from the Mongoose sibling project, a single injected object that gives you a full set of typed database operations for one specific kind of stored data, just with a different vocabulary of methods (`.find()`, `.findOneBy()`, `.save()`, `.delete()`, `.create()` here, versus Mongoose's `.find()`, `.findById()`, `.findByIdAndUpdate()`, `.findByIdAndDelete()` there).

`@InjectRepository(Employee)` is the parameter decorator that makes this possible, the direct equivalent of `@InjectModel(Student.name)` from the sibling project. Normally NestJS figures out what to inject purely from a constructor parameter's TypeScript type, but that trick does not work here because `Repository<Employee>` is a generic type from the `typeorm` library itself, not a unique class this project defines, so there is no unique class for Nest to match against automatically. `@InjectRepository(Employee)` resolves that ambiguity explicitly, telling Nest to look up the specific provider that `TypeOrmModule.forFeature([Employee])` registered back in the module.

## The CRUD methods, one by one

```ts
async create(employeeData: Partial<Employee>): Promise<Employee> {
    const employee = this.employeeRepository.create(employeeData);
    return this.employeeRepository.save(employee);
}
```

`this.employeeRepository.create(employeeData)` does not touch the database at all, despite the name, it just builds a new, plain `Employee` instance in memory out of whatever fields `employeeData` provides. `this.employeeRepository.save(employee)` is the call that actually issues an `INSERT` statement to Postgres and returns the saved row, now including its database generated `id`. This two step create then save pattern is standard TypeORM style, and it mirrors the two step feel of Mongoose's `new StudentModel({...})` followed by `.save()` from the sibling project, just phrased through a repository object instead of a model constructor.

```ts
async findAll(): Promise<Employee[]> {
    return this.employeeRepository.find();
}
```

`.find()` with no arguments returns every row in the `employee` table, the direct equivalent of Mongoose's bare `.find()`.

```ts
async findOne(id: number): Promise<Employee> {
    const employee = await this.employeeRepository.findOneBy({ id });
    if(!employee) {
        throw new NotFoundException(`Employee with ID ${id}not found`);
    }
    return employee;
}
```

`.findOneBy({ id })` looks up a single row by an exact match on its `id` column, TypeORM's equivalent of Mongoose's `.findById()`. If nothing matches, `findOneBy` resolves to `null` rather than throwing, so the `if(!employee)` check and the `throw new NotFoundException(...)` are what turn a missing row into a proper `404 Not Found` HTTP response, this is exactly the same defensive pattern the MongoDB sibling project's CRUD note recommends and it is genuinely present here.

There is a real, small bug worth flagging honestly in that error message though: `` `Employee with ID ${id}not found` `` is missing a space between the interpolated `${id}` and the word `not`. Calling `findOne(5)` on a missing row would produce the message "Employee with ID 5not found" rather than "Employee with ID 5 not found". It does not break the `404` behavior itself, the exception still throws correctly, it is purely a cosmetic typo in the message text.

```ts
async update(id: number, updatedData: Partial<Employee>): Promise<Employee> {
    const employee = await this.employeeRepository.findOneBy({ id });
    if(!employee) {
        throw new NotFoundException(`Employee with ID {id} not found`);
    }
    const updated = Object.assign(employee, updatedData);
    return this.employeeRepository.save(updated);
}
```

This method first looks up the existing row the same way `findOne` does, then uses `Object.assign(employee, updatedData)` to merge whatever fields `updatedData` supplies onto the existing, fully loaded `employee` object (any field not present in `updatedData` is left untouched), then calls `.save()` again, which TypeORM is smart enough to turn into an `UPDATE` statement rather than a fresh `INSERT`, since the object already has an `id`.

There is a second, more real bug here, distinct from the first one: `` `Employee with ID {id} not found` `` is missing the dollar sign in front of `{id}`. In a JavaScript template literal, only `${expression}` gets interpolated, plain `{id}` is just literal text. This means if `update()` is called with a nonexistent id, the thrown error message is literally the string "Employee with ID {id} not found", the actual id value never appears in it at all, unlike the `findOne` typo above where the id at least shows up (just without a following space), here it does not show up whatsoever. Both are genuine, verifiable bugs sitting a few lines apart in the same file, and both are the kind of small mistake that is easy to miss in review since the code still runs and still throws the right HTTP status code, only the message text is wrong.

```ts
async delete(id: number): Promise<{ message: string }> {
    const result = await this.employeeRepository.delete(id);

    if(result.affected === 0) {
        throw new NotFoundException(`Employee with ID ${id} not found`);
    }
    return { message: `Employee with ID ${id} has been deleted successfully!`}
}
```

`.delete(id)` issues a `DELETE` statement directly, without first loading the row, and returns a result object whose `affected` property tells you how many rows were actually deleted. Checking `result.affected === 0` is how this method detects "nothing matched that id" without needing a separate lookup first, a slightly different but equally valid approach compared to `findOne` and `update`'s look then act pattern. Both interpolations in this method are written correctly, `${id}` appears properly both in the not found message and in the success message.

## `search()`, the one method that uses TypeORM's query builder

```ts
async search(filters: { name?: string; department?:string}): Promise<Employee[]> {
    const query = this.employeeRepository.createQueryBuilder('employee');

    if(filters.name){
        query.andWhere('employee.name ILIKE :name', {name: `%${filters.name}%`});
    }
    if(filters.department){
        query.andWhere('employee.department = :dept', {dept: filters.department});
    }
    return query.getMany();
}
```

`.createQueryBuilder('employee')` starts building a raw SQL query by hand, using TypeORM's fluent query builder API, rather than the simpler `.find()`/`.findOneBy()` helpers used everywhere else in this file. This is the tool you reach for once your query has conditions that only apply sometimes, exactly the situation here, both `name` and `department` are optional filters. `.andWhere('employee.name ILIKE :name', { name: \`%${filters.name}%\` })` only gets added to the query at all if `filters.name` was actually provided, and it uses Postgres's `ILIKE` operator, a case insensitive pattern match, wrapping the search term in `%` wildcards so it matches anywhere inside the name, not just an exact match. `.andWhere('employee.department = :dept', { dept: filters.department })` is a plain exact match on department, added only if that filter was provided. `:name` and `:dept` are named parameter placeholders, TypeORM safely substitutes the actual values in a parameterized way, avoiding SQL injection, which is the entire reason to use a query builder's parameter binding rather than concatenating raw strings by hand. `.getMany()` finally executes the built query and returns every matching row as an array of `Employee` objects.

One thing worth being precise about, since it is easy to assume otherwise: if `search()` is called with neither `name` nor `department` supplied, neither `if` branch adds a condition, and `query.getMany()` simply returns every employee in the table, the same as `findAll()` would, just via a different code path.
