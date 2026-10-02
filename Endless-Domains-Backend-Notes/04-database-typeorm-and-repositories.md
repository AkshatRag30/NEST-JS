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

## Update from the October 2026 uat pull

Between commit `a131b429` and `uat` merge commit `dc1ba3e8`, marketplace v2 added six new tables, and how they reach the database says something important about this team's migration workflow. One version fact first: `package.json` on `uat` pins `"typeorm": "^0.3.17"`, so this is TypeORM 0.3 with the `DataSource` API, not the 0.2 listed in the project CLAUDE.md. The `DataSourceOptions` type in the excerpt above is the giveaway.

### The six new tables

| Table | Entity file | What a row is |
|---|---|---|
| `tbl_marketplacev2_orders` | `marketplacev2/order/entity/order.entity.ts` | one signed Seaport listing: order hash, signature, the full order parameters, price, status (active, filled, cancelled, expired, invalid) and who filled it |
| `tbl_marketplacev2_watchlist` | `marketplacev2/order/entity/watchlist.entity.ts` | one user watching one token, unique on user plus token contract plus token id |
| `tbl_marketplacev2_listing_status` | `marketplacev2/listing-status/entity/listing-status.entity.ts` | the current listed or unlisted state of a domain for a user, so the "my domains" view does not need to recompute it |
| `tbl_marketplacev2_listing_history` | `marketplacev2/listing-status/entity/listing-history.entity.ts` | an append only log of listing events (listed, cancelled, expired, sold, invalidated), unique on order hash plus event type |
| `tbl_marketplacev2_transactions` | `marketplacev2/transaction/entity/transaction.entity.ts` | one completed sale, with buyer, seller, price and transaction hash, written by the poller |
| `tbl_marketplacev2_poller_cursor` | `marketplacev2/poller/entity/poller-cursor.entity.ts` | a single row (id 1) recording the last block the event poller has fully processed |

The orders table is the first in this codebase to use a partial unique index, `idx_marketplacev2_orders_active_listing_unique` on `(tokenContract, tokenId)` where the status is active. It is worth understanding because it is such a clean idea. It makes the database itself guarantee that one token can never have two active listings at once, while still allowing any number of old cancelled, expired or filled orders for the same token. When two create requests race, the loser hits a unique violation, and the service maps that error to an `ALREADY_LISTED` rejection instead of a 500. That is the database enforcing a business rule that application code alone could not enforce safely under concurrency. Compare it with the v1 buy flow in [cluster 03](clusters/03-commerce-and-marketplace/03-domain-listing-and-buy-domain-secondary-marketplace.md), which used an atomic conditional `UPDATE` for the same kind of job. Each table is described column by column in [cluster 12](clusters/12-marketplace-v2/00-README.md), notes 03 and 07.

Several of these entities write `createdAt` and similar times as `timestamp` columns without a time zone, and cursors are built from them with `toISOString()`. On a server whose clock is not set to UTC, or a developer laptop in India writing to the shared UAT database, that combination can skip or repeat rows across pages and confuse the poller's stale event checks. Notes 06 and 09 in cluster 12 explain exactly where. As a rule, prefer `timestamptz` for anything that represents a real moment in time.

### No migration files, and why the tables still appear

You might expect six new entities to come with six new migration files. The diff contains none, and that is not an oversight in this update, it is how this team deploys. `run/deploy.sh` does this on every deploy:

```sh
# run/deploy.sh
rm -r src/migration
npm run typeorm:migration:generate -n "update"
npm run typeorm:migration:run
npm run seed:run
node dist/main.js
```

In other words, migrations are not kept in version control at all. Every deploy deletes the folder, asks TypeORM to compare the entity classes against the live database, generates one fresh migration called `update` containing whatever differs, and runs it. That is why `synchronize: false` and "no committed migrations" can both be true while new tables still appear after a deploy.

This approach has real costs, and you should understand them before you rely on it. Nobody reviews the generated SQL before it runs against a real database. If a column is renamed in an entity, TypeORM's diff sees one column disappear and another appear, so the generated migration drops the old column with its data and adds an empty new one. There is no history of schema changes in git, so you cannot see when a column arrived or roll one change back cleanly. And two environments deployed from the same commit can end up with different migrations if their databases had drifted. The conventional alternative, generating a migration locally, reading it, committing it, and having deploy only run migrations, is what the `typeorm:migration:generate` and `:run` scripts were designed for. If you add an entity yourself, it is worth asking the team which of the two they actually want you to follow.
