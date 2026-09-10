# 04. Blog Post View Tracking

## This is not a blog CMS, and that is worth sitting with for a moment

The task brief for this cluster described `blog` as "a real content management system for a company blog" and asked to look for how posts, categories, or authors are modeled. Having read every file in the folder, none of those exist here. There is no `Post` entity, no `Category` entity, no `Author` entity, no controller that creates or edits a blog post. The two entities that do exist, `PostView` and `ViewEvent`, both key everything off a plain string `slug`, which means the actual blog posts, their titles, bodies, and publication dates, are managed somewhere entirely outside this backend, most likely a separate headless CMS or a statically generated frontend that this API never touches. What `blog` actually is, in full, is a small, self contained analytics service that counts how many times each post slug has been viewed. This is exactly the kind of correction the task description asked to verify rather than assume, and it turned out the folder name was misleading on its own.

## Two tables doing two different jobs

```ts
@Entity('tbl_post_views')
export class PostView {
    @PrimaryColumn({ type: 'varchar' }) slug: string;
    @Column({ type: 'bigint', default: 0 }) total_views: string;
    @Column({ type: 'timestamptz', default: () => 'now()' }) updated_at: Date;
}
```

`tbl_post_views` is the simple, fast to read aggregate, one row per slug holding a running total.

```ts
@Entity('tbl_view_events')
@Unique('uq_view_events_daily', ['slug', 'ipHash', 'viewDate'])
export class ViewEvent {
    @PrimaryGeneratedColumn('increment', { type: 'bigint' }) id: string;
    @Column({ type: 'varchar' }) slug: string;
    @Column({ type: 'varchar', name: 'ip_hash' }) ipHash: string;
    @Column({ type: 'date', name: 'view_date', default: () => 'CURRENT_DATE' }) viewDate: Date;
    @Column({ type: 'timestamptz', name: 'viewed_at', default: () => 'now()' }) viewedAt: Date;
}
```

`tbl_view_events` is the raw, deduplicated event log, one row per unique combination of slug, hashed visitor IP, and calendar day, enforced by a real unique constraint rather than application logic. This table is what makes counting honest, a visitor refreshing the same post fifty times in one day only ever produces one row, and it is also what powers the "most popular this week or month" query, since it can be grouped and counted by date range in a way the plain running total cannot.

## Recording a view, without ever storing a real IP address

```ts
export function hashIp(clientIp: string, secretKey: string, date: Date = new Date()): string {
    const salt = getDailySalt(secretKey, date);
    return createHash('sha256').update(clientIp + salt).digest('hex');
}
```

The salt itself rotates every calendar day (`secretKey + the UTC date`, hashed), which produces a small but genuinely clever property, the same visitor IP hashes to the same value within one day, which is exactly what the unique constraint needs to deduplicate repeat views, but hashes to a completely different, unrelated value the next day, so nothing in this table can be used to track one visitor's activity across days, and the real IP address itself is never written to disk anywhere. `ViewsService.onModuleInit` refuses to start at all if the salt key configured in the environment is shorter than thirty two characters, so a weak secret cannot silently ship.

`ViewsRepository.recordView` does the insert and the counter bump as two statements inside one query runner, using `ON CONFLICT ... DO NOTHING` on the event insert (so a duplicate same day view is silently ignored) and only bumping `tbl_post_views.total_views` when that insert actually succeeded, which is how the running total and the deduplicated event log stay in agreement with each other.

## The public surface, and what is actually protected

`ViewsController` answers at `/blog`. Recording a view (`POST /blog/views/:slug`) is rate limited to ten calls per minute per client through `ThrottlerGuard`, since this route accepts traffic from any anonymous visitor's browser. Reading batch view counts for a listing page (`GET /blog/views?slugs=a,b,c`) and reading the popular posts widget (`GET /blog/popular`) are both open reads with no rate limit, since they are cheap lookups a page can call on every load. The one genuinely protected route is `POST /blog/admin/prune`, gated by `AccessTokenGuard` and `SuperAdminAccessGuard`, which lets a superadmin manually trigger the same cleanup the nightly cron job runs on its own.

## The nightly prune job, and a real Postgres constraint it works around

```ts
@Cron('0 2 * * *', { name: 'pruneOldViewEvents', timeZone: 'UTC' })
async pruneOldViewEvents(): Promise<void> {
    if (process.env.NODE_ENV === 'test') return;
    ...
}
```

Every night at 2am UTC, `PruneService` deletes `ViewEvent` rows older than thirty days (the running totals in `PostView` are untouched, only the detailed per visitor log is pruned). The delete itself is worth reading because Postgres genuinely has no `DELETE ... LIMIT`, so a single unbounded delete against a table with months of accumulated rows risks locking the table for a long time. The repository works around this with a loop that deletes in batches of ten thousand rows using a subquery, stopping once a batch comes back empty. This is a real, general purpose pattern worth recognizing anywhere a cleanup job needs to touch a large table without one giant transaction.
