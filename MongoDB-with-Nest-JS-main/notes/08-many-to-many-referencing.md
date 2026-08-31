# 08. Many to Many Referencing

The last relationship shape in this repo is many to many, demonstrated by the `project` module: a `Developer` can work on many `Project`s, and a `Project` can have many `Developer`s. Both sides of the relationship hold an array of references to the other side, which is what makes this different from the one to many case in the previous note, where only the "one" side (the library) held the array, while books had no reference back to their library at all.

## The two schemas, and a real bug worth noticing

```ts
// src/project/schemas/project.schema.ts
@Schema({ timestamps: true })
export class Project extends Document {
    @Prop({ required: true })
    title: string;

    @Prop({ type: [{ type: Types.ObjectId, ref: 'Developer'}]})
    developers: Types.ObjectId
}
export const ProjectSchema = SchemaFactory.createForClass(Project);
```

```ts
// src/project/schemas/developer.schama.ts
@Schema({ timestamps: true })
export class Developer extends Document {
    @Prop({ required: true })
    name: string;

    @Prop({ type: [{ type: Types.ObjectId, ref: 'Project'}]})
    projects: Types.ObjectId
}
export const DeveloperSchema = SchemaFactory.createForClass(Developer);
```

The `@Prop({ type: [{ type: Types.ObjectId, ref: 'Developer' }] })` decorator on `Project.developers` is the exact same array-of-references pattern as `Library.books` from the previous note, so at the actual Mongoose schema level (what gets sent to MongoDB and what queries actually operate on), `developers` really is an array of ObjectIds, and the code works correctly at runtime, as you can see in `project.service.ts` below, where whole arrays of ObjectIds get assigned to it without any error.

What is worth flagging honestly, though, is the TypeScript type written right after the property name: `developers: Types.ObjectId` and `projects: Types.ObjectId`, both singular, not `Types.ObjectId[]`. This is a mismatch between the `@Prop()` decorator's runtime configuration (an array) and the TypeScript type annotation sitting right next to it (a single value). It happens to cause no actual failures in this repo, because `noImplicitAny: false` is set in `tsconfig.json` (see [01-project-setup-and-bootstrap.md](01-project-setup-and-bootstrap.md)) and because the two spots in `project.service.ts` that write to these fields do so through the loosely typed `.create({...})` and `findByIdAndUpdate(..., { $set: {...}})` calls rather than direct property assignment that TypeScript would type-check strictly against `Project`/`Developer`. If you tried to write `someProject.developers = [dev1._id, dev2._id]` as a direct assignment on a typed variable, though, TypeScript would flag it as an error, since you would be assigning an array to a property typed as a single `ObjectId`. This is a small but genuine inconsistency in the repo's code, worth noticing precisely because it demonstrates that a `@Prop()` decorator's configuration and its neighboring TypeScript type annotation are two independent things that you, the developer, are responsible for keeping in sync, Mongoose and TypeScript do not cross check them against each other automatically.

## Seeding both sides of the relationship: `project.service.ts`

```ts
async seed(): Promise<{ dev1: Developer; dev2: Developer }> {
    const [projectA, projectB] = await Promise.all([
        this.projectModel.create({ title: 'Nest CRM'}),
        this.projectModel.create({ title: 'Mongo Analytics'})
    ]);

    const [dev1, dev2] = await Promise.all([
        this.developerModel.create({
            name: 'Farzeen',
            projects: [projectA._id, projectB._id],
        }),
        this.developerModel.create({
            name: 'Huzaifa',
            projects: [projectA._id],
        })
    ])

    await Promise.all([
        this.projectModel.findByIdAndUpdate(projectA._id, {
            $set: { developers: [dev1._id, dev2._id]}
        }),
        this.projectModel.findByIdAndUpdate(projectB._id, {
            $set: { developers: [dev1._id]}
        })
    ]) 
    return { dev1, dev2 };
}
```

This method, reachable through `POST /project/seed`, builds out a small, realistic many to many dataset in three stages, and is worth reading closely because it shows exactly why many to many referencing needs more setup work than one to one or one to many.

Stage one creates two `Project` documents, `projectA` (Nest CRM) and `projectB` (Mongo Analytics), with `Promise.all([...])` running both `create()` calls concurrently rather than one after another, since they do not depend on each other. Neither project has any developers listed yet, that field is left undefined at this point.

Stage two creates two `Developer` documents, and this is where the first half of the relationship gets written: Farzeen is created with `projects: [projectA._id, projectB._id]` (assigned to both projects), and Huzaifa is created with `projects: [projectA._id]` (assigned to only the first one). At this point, if you queried the `developers` collection, both developers would correctly list their projects, but if you queried the `projects` collection, neither project would list any developers yet, the relationship is only half written.

Stage three fixes that asymmetry by going back and updating each `Project` document directly, using `findByIdAndUpdate` with `$set: { developers: [...] }`. `$set` is a raw MongoDB update operator (not a Mongoose specific concept, this is the same `$set` you would use with the MongoDB shell or driver directly) meaning "set this field to this new value, leave every other field on the document untouched," which is exactly what is wanted here, only the `developers` field should change, `title` should be left alone. `projectA` gets both developer ids set, since both Farzeen and Huzaifa are assigned to it, while `projectB` only gets Farzeen's id.

This three stage dance, create both sides independently, then go back and update the second side to point back at the first, is the direct, hands on cost of many to many referencing that the one to many `Library`/`Book` example did not have: with many to many, both collections need to be kept in sync with each other, since MongoDB has no built in concept of a relationship "table" the way a SQL database's join table works, keeping both sides consistent is entirely the application code's responsibility.

## Reading the relationship back: `.populate()` combined with `.lean()`

```ts
async getDevelopers(): Promise<Developer[]>{
    return this.developerModel.find().populate('projects').lean();
}
async getProjects(): Promise<Project[]>{
    return this.projectModel.find().populate('developers').lean();
}
```

`.populate('projects')` and `.populate('developers')` work exactly as explained in the previous note, turning arrays of raw ObjectIds back into arrays of full documents. The new piece here is `.lean()`, which is not used anywhere else in this repo. Normally, a Mongoose query resolves with full Mongoose document instances, objects with extra behavior attached like `.save()` and internal change tracking machinery, which has a real performance cost to construct. `.lean()` tells Mongoose to skip building those full document instances and instead resolve with plain, ordinary JavaScript objects, which is faster and uses less memory. This is a sensible choice specifically for these two read-only endpoints, since neither `getDevelopers` nor `getProjects` ever calls `.save()` on what comes back, there is no reason to pay for Mongoose document features that will never be used, this is exactly the kind of "read heavy" case the referencing tradeoffs discussion in [10-relationship-concepts-from-lecture.md](10-relationship-concepts-from-lecture.md) has in mind.

## The controller

```ts
@Controller('project')
export class ProjectController {
    constructor(private readonly service: ProjectService){}

    @Post('seed')
    seedData(){
        return this.service.seed();
    }
    @Get('developers')
    getDevelopers(){
        return this.service.getDevelopers();
    }
    @Get()
    getProject(){
        return this.service.getProjects();
    }
}
```

Three routes: `POST /project/seed` triggers the whole three stage seeding process above, `GET /project/developers` lists every developer with their projects populated, and `GET /project` (note there is no `getProject`/singular route with an `:id`, despite the method being named `getProject`, it actually returns the full list of every project) lists every project with their developers populated.
