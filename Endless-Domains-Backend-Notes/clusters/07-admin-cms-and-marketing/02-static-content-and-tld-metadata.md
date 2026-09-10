# 02. Static Content and TLD Metadata

## Two folders, two very differently protected content stores

`content-managment` and `tlds-content-managment` both do the same basic job, letting someone edit content that a marketing page will later read, but they were clearly built by different hands or at different times, because their approach to who is allowed to write to them is completely different. Reading both side by side is a good exercise in noticing that a folder name promising similar responsibility does not guarantee similar care.

## `content-managment`: generic key and value page content

`StaticContentEntity` is about as simple as an entity gets:

```ts
@Entity({ name: 'tbl_static_content' })
export class StaticContentEntity extends BaseEntity {
    @Column({ nullable: false, type: 'varchar', length: 250 })
    title: string;
    @Column({ nullable: false, type: 'text' })
    content: string;
    @Column({ nullable: false, type: 'varchar', length: 50, unique: true, comment: 'A unique key to identify content type (e.g., "terms", "privacy", "about")' })
    contentKey: string;
}
```

One row per static page, looked up by a short, unique `contentKey` like `terms` or `privacy` or `about`. `StaticContentController` exposes create, update, fetch by key, and delete, all under `/static-content`. This is the entire feature.

The finding worth flagging clearly here is that not one of these four routes carries a `@UseGuards(...)` decorator, not even the ones that create, overwrite, or delete a page's entire content. Every other write heavy admin controller looked at in this cluster puts at least one guard in front of mutating routes, most commonly `AccessTokenGuard`. This one puts none. Whether that is deliberate (maybe this route only exists behind a network boundary or an API gateway that this codebase itself never shows) or a genuine oversight is not something the code itself answers, but as written, anyone who can reach this API can rewrite the site's terms of service page.

## `tlds-content-managment`: the content behind each TLD's own landing page

`TldsJsonEntity` is a different shape entirely, a `tld` string, a `jsonb` `details` column, and an `isPublished` boolean:

```ts
@Entity({ name: 'tbl_tlds_json' })
export class TldsJsonEntity extends BaseEntity {
    @Column({ type: 'varchar', length: 255, unique: true, nullable: false })
    tld: string;
    @Column({ type: 'jsonb', nullable: false })
    details: TldDetails;
    @Column({ type: 'boolean', default: false })
    isPublished: boolean;
}
```

`TldDetails` (in `interface/tlds-details.interface.ts`) is a whole structured page in one JSON blob, a hero section with a headline and subheadline, a stats block, an about block, a why block, an unlocks block, and a FAQ block, plus SEO metadata (title, description, keywords). This is the data that renders each TLD's own marketing landing page, the kind of page a domain marketplace would show at something like `/tlds/eth`, and the `isPublished` flag lets an admin stage a new TLD's page before it goes live.

Unlike `content-managment`, the write routes here are actually locked down, `create`, `update`, and `delete` all require `SuperAdminAccessGuard`, while the two read routes, fetching one TLD's content by name and listing all TLDs with pagination, search, and an `isPublished` filter, are left open for the public marketing site to read. That split, public reads, superadmin writes, is the pattern this cluster's other content folders arguably should have followed too.

## The locale aware read path is the most interesting code in this file

`getTldsJsonByTld` does not just fetch a row:

```ts
async getTldsJsonByTld(tld: string, locale?: string): Promise<TldsJsonEntity> {
    const record = await this.tldJsonRepo.findTld(tld);
    if (!record) throw new NotFoundException('TLD JSON record not found');
    const resolvedLocale = resolveLocale(locale);
    if (resolvedLocale === 'en') return record;
    const cacheKey = buildTranslationCacheKey('tlds-json', tld, resolvedLocale, record.lastChangedDateTime);
    const cached = await this.translationCache.get<TldsJsonEntity>(cacheKey);
    if (cached) return cached;
    const translatedDetails = await this.translationService.translateObjectFields(
        record.details, TLD_TRANSLATABLE_FIELD_PATHS, resolvedLocale,
    );
    if (translatedDetails === record.details) return record;
    const localized = { ...record, details: translatedDetails };
    await this.translationCache.set(cacheKey, localized, TRANSLATION_CACHE_TTL_MS);
    return localized;
}
```

If the caller asks for a non English locale, this checks a translation cache first, and if nothing is cached, it calls out to a real translation service (covered in full in [03-localization-and-translation.md](03-localization-and-translation.md)) and caches the translated result for a week. `TLD_TRANSLATABLE_FIELD_PATHS` is a hand maintained list of exactly which dot paths inside the JSON blob hold actual prose worth translating (headlines, descriptions, FAQ question and answer text) versus which fields are data that must never be touched (the stat's numeric looking `value` string, image URLs, the TLD name itself, booleans). This means every single TLD page can be served in any of the eight supported locales without a translator ever having manually typed a second copy of the page, the translation happens live and gets cached rather than being pre-written.

## Why these two folders end up in one file

They are grouped together here because they answer the same kind of question, what does a marketing page look like, but they are separate, independently deployed pieces of code with no shared entity, service, or controller, and it is worth remembering they solve genuinely different problems, one flat unstructured page per key versus one richly structured page per TLD with built in localization.
