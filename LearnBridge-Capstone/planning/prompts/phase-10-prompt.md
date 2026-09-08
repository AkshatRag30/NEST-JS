# Phase 10 Prompt: TypeORM Comparison Appendix (optional)

This phase is optional and, unlike every other phase, deliberately does not touch the main LearnBridge project at all. Use this whenever you want the comparison, not necessarily right after phase 9.

Copy everything inside the fenced block below into Claude Code as one message.

```
Before writing any code, read planning/14-phase-10-PRD-typeorm-comparison-appendix.md completely, which lives inside the LearnBridge-Capstone project you have been working in, but the work for this phase must not happen inside that project.

Create a brand new, separate sibling folder called LearnBridge-TypeORM-Appendix, one level up from LearnBridge-Capstone, and scaffold a fresh, minimal NestJS project inside it, entirely independent of the main LearnBridge codebase.

Install @nestjs/typeorm, typeorm, and pg. Build a Category entity using @Entity, @Column, and @PrimaryGeneratedColumn, with an id and a unique name. Build a Course entity the same way, with an id, a title, a description, a price, and a @ManyToOne relation to Category. Connect to a real local Postgres database using TypeOrmModule.forRoot, with synchronize enabled since this is a throwaway learning project and a real migration workflow is not the point here. Build a small CategoriesModule and CoursesModule, each with a service using @InjectRepository to get a Repository for its entity, and a controller exposing basic create and list endpoints, when listing courses, use the relations option to include each course's category in the response.

Once this is working, confirmed by actually creating a category, creating a course under it, and fetching that course back with its category attached, write a short markdown file at the root of LearnBridge-TypeORM-Appendix called comparison-notes.md. In it, write in your own plain words, based on what you actually just built, not a generic summary pulled from documentation, the concrete differences you noticed between this and the Prisma based Course and Category setup already sitting in the main LearnBridge project's phase 3, specifically covering whether you needed a real migration step or could rely on synchronize, how calling a TypeORM repository's find and save methods felt compared to calling PrismaService.course.findMany and PrismaService.course.create, and whether the @Entity and @Column decorators felt closer to or further from the @Schema and @Prop decorators from the Mongoose notifications module in the main project.

When you are done, show me the comparison notes file in full, and tell me one specific thing you would genuinely reach for TypeORM over Prisma for, and one specific thing you would genuinely reach for Prisma over TypeORM for, based on your own hands on experience just now, not a generalization.
```
