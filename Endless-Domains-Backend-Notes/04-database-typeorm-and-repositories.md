# 04. Database, TypeORM, and the Base Repository

## A different ORM than the MongoDB or Prisma reference projects

This app uses TypeORM, the same ORM from the `PostgreSQL-with-NEST-JS-main` reference project, not Prisma and not Mongoose, and it uses it the way a real, multi year production system does, with real migrations and a database connection assembled from secrets rather than a plain connection string.

## `TypeOrmConfigService`, assembled from secrets, not a `.env` file

```ts
// src/@core/config/type-orm-config.service.ts
public async createTypeOrmOptions(): Promise<DataSourceOptions> {
    await this.ensureValues(['POSTGRES_HOST', 'POSTGRES_PORT', 'POSTGRES_USER', 'POSTGRES_PASSWORD', 'POSTGRES_DATABASE']);
    return {
        type: 'postgres',
        host: this.getValue('POSTGRES_HOST'),
        port: parseInt(this.getValue('POSTGRES_PORT')),
        username: this.getValue('POSTGRES_USER'),
        password: this.getValue('POSTGRES_PASSWORD'),
        database: this.getValue('POSTGRES_DATABASE'),
        entities: [
            join(__dirname, './common/**', '*.entity.{ts,js}'),
            join(__dirname, '../../components/**', '*.entity.{ts,js}'),
            join(__dirname, '../../components/**/**', '*.entity.{ts,js}')
        ],
        migrationsTableName: 'migration',
        migrations: [join(__dirname, '../../migration', '*.{ts,js}')],
        synchronize: false,
        ssl: this.isProduction(),
    } as DataSourceOptions;
}
```

`ensureValues` calls `SecretsService.getSecret` under the hood (through its own small private `fetchSecrets` method, a second, separate implementation of the exact same "fetch and cache a secret bundle" idea already covered in the configuration note, rather than reusing the shared `SecretsService` class directly, worth noticing as another example of the same small duplication pattern), and only once those five Postgres values are confirmed present does it build the actual connection options object.

The `entities` array is worth understanding well, since it explains something that would otherwise be confusing about this codebase's structure, no single file anywhere lists every entity in the app. Instead, TypeORM is told to glob, at startup, for any file ending in `.entity.ts` sitting inside `src/@core/common` or anywhere inside `src/components`, however deeply nested, and treat every one of them as a table definition. This is why a brand new feature module can add a new entity file anywhere inside its own folder and have it picked up automatically, with no central registry file to remember to update, a real convenience at this scale, but also why finding "every entity in the system" requires a search across the whole `src/components` tree rather than opening one file.

`synchronize: false` is the single most important production safety setting on this whole object. In development, TypeORM (and Prisma's `db push`, and TypeORM's own `synchronize: true` option) can happily rewrite your database's actual table structure to match your entity classes automatically, convenient while learning, genuinely dangerous against a real database with real user data, since an automatic sync can silently drop or alter a column in a way that loses data. `synchronize: false` means the only way this database's schema ever changes is through an explicit, reviewed migration file, generated with `npm run typeorm:migration:generate` and applied with `npm run typeorm:migration:run`, exactly the disciplined workflow the `PostgreSQL-with-NEST-JS-main` reference project's own notes already touched on conceptually. `ssl: this.isProduction()` is a similarly real detail, the database connection is encrypted in production and not necessarily in local development, a common, sensible tradeoff.

## The generic base repository, and a question worth answering yourself

```ts
// src/repositories/base/base.abstract.repository.ts
export class BaseAbstractRepository<T> implements BaseInterfaceRepository<T> {
    private entity: Repository<T>;
    protected constructor(entity: Repository<T>) { this.entity = entity; }
    public async save(data: T | any): Promise<T> { return await this.entity.save(data); }
    public async update(id: string, data: any): Promise<UpdateResult> { return await this.entity.update(id, data); }
    public async findOneById(id: string): Promise<T> { return await this.entity.findOne({ where: { id } as any }); }
    public async findAll(): Promise<T[]> { return await this.entity.find(); }
    public async remove(id: string): Promise<DeleteResult> { return await this.entity.delete(id); }
}
```

This is a small, generic wrapper around a TypeORM `Repository<T>`, offering the five most common operations, save, update, find one by id, find all, and delete, through one reusable abstract class any feature's own repository could extend instead of writing those same five methods over and over. It is a real, sensible pattern, and it is also exactly the kind of thing worth verifying empirically rather than assuming is used everywhere just because it exists, once you are reading actual feature modules in the cluster notes, keep an eye on whether a given module's service extends this base repository, or whether it simply injects a plain TypeORM `Repository<SomeEntity>` directly with `@InjectRepository` and calls methods on it straight away, skipping this abstraction entirely. Both are completely valid ways to use TypeORM, and a codebase built by many people over several years very often ends up with both styles living side by side, which one you find in a given module tells you something real about when that module was likely written and by whom, a genuinely useful piece of situational awareness in any large, real codebase, not just this one.

## Seeding

`src/db/seeding/seeds` holds seed files (`create-domain-provider-and-tld.seed.ts`, `create-domain-provider-smart-contracts.seed.ts`, `create-custom-domain-records.seed.ts`), run through the `typeorm-extension` package's own seeding CLI (`npm run seed`), populating baseline reference data, which domain providers and top level domains (TLDs) exist, which smart contracts back which chains, and default custom domain records, the kind of foundational lookup data an app needs present before any real user activity can happen against it. Reading these three files directly is a fast, low risk way to see several real entities' shapes at once without needing to trace through a live feature flow first.
