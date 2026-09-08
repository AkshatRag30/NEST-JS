# Phase 10 PRD: TypeORM Comparison Appendix (optional)

## Goal

This phase is optional and does not add anything to the finished product, its entire purpose is comparison, so that TypeORM, the ORM used in `PostgreSQL-with-NEST-JS-main`, gets built with your own hands at least once rather than only being read about, and so you can feel directly what is actually different between it and Prisma, the ORM the rest of LearnBridge is built on.

## Concepts practiced

`@Entity`, `@Column`, `@PrimaryGeneratedColumn`, `@ManyToOne` and `@OneToMany` relation decorators, `TypeOrmModule.forRoot` and `forFeature`, the repository pattern through `@InjectRepository`, and a direct, side by side comparison against the Prisma schema and client calls already used everywhere else in this project.

## Scope

Do not touch the main LearnBridge codebase for this phase, build it in a small, throwaway sibling project instead, the same way the original eight reference projects were each their own separate folder. Rebuild exactly one piece of LearnBridge's data model with TypeORM instead of Prisma, the Category and Course entities and their relationship, nothing more, this is intentionally the smallest possible slice that still has a real one to many relationship in it. Write the `Category` and `Course` entities with `@Entity()`, `@Column()`, and `@PrimaryGeneratedColumn()`, connect `Course` to `Category` with `@ManyToOne`, and expose a small controller and service using `@InjectRepository(Course)` and its `find`, `findOneBy`, `save`, and `delete` methods.

Once it works, write down, in your own words, in a short markdown file next to that throwaway project, the concrete differences you actually felt while building it. Specifically, whether you needed to write and run a migration or whether `synchronize: true` handled it during development, how `@InjectRepository` compares to injecting a single `PrismaService` and calling `prisma.course.findMany()`, and whether the entity decorators felt closer to or further from the Mongoose `@Schema`/`@Prop` decorators you already know well.

## Acceptance criteria

1. The throwaway project boots, connects to a real Postgres database, and can create a category, create a course under it, and fetch the course back with its category included, using `relations: ['category']` on the repository call.
2. You can explain, without looking anything up, the one sentence difference between a TypeORM repository call and a Prisma client call for the same operation.
3. Your written comparison notes name at least one thing you genuinely preferred about each approach, not just a restatement of the official marketing for either tool.

## Explicit trap to avoid

Do not skip actually writing this phase's small project and only read about the difference. Reading `PostgreSQL-with-NEST-JS-main`'s notes again is not the same experience as watching your own `@ManyToOne` decorator either work or fail to compile, and the entire value of this appendix is in the second thing, not the first.
