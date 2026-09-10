# 03. Localization and Translation

## Correcting the assumption before anything else

The most natural guess for a folder named `localization` is that it holds translated strings, a set of files with one entry per language the way a typical i18n setup works. That is not what this is. `src/components/localization` holds a small, self contained service that calls out to Azure's real Cognitive Services Translator API at the moment content is requested, translates only the specific fields worth translating, and caches the result. There is no translated content sitting at rest anywhere in this codebase, it is generated on demand and then remembered for a while.

## The supported locales and a real naming mismatch worth knowing about

```ts
export const SUPPORTED_LOCALES = ['en', 'hi', 'es', 'zh-CN', 'fil', 'nl', 'id', 'ar'] as const;

export const LOCALE_TO_AZURE_LANGUAGE_CODE: Record<SupportedLocale, string> = {
    en: 'en', hi: 'hi', es: 'es', 'zh-CN': 'zh-Hans', fil: 'fil', nl: 'nl', id: 'id', ar: 'ar',
};
```

English, Hindi, Spanish, Simplified Chinese, Filipino, Dutch, Indonesian, and Arabic. Every locale code the rest of the codebase uses matches Azure's own language code directly except one, Simplified Chinese, where Azure expects `zh-Hans` rather than `zh-CN`. This little lookup table exists entirely to paper over that one mismatch, and it is a good example of the kind of detail that looks trivial until it silently breaks one specific language for one specific vendor's API.

`resolveLocale` is the gate every caller goes through, if the requested locale is not in that list, it throws a `BadRequestException` naming every locale that is actually supported, rather than silently falling back to English or to whatever Azure happens to guess.

## The translation itself only ever touches prose, nothing else

`TranslationService.translateObjectFields` is the core piece of logic, and it is worth reading closely because of how carefully it protects data that is not meant to be translated:

```ts
async translateObjectFields<T>(obj: T, fieldPaths: string[], targetLocale: SupportedLocale): Promise<T> {
    if (obj === null || obj === undefined || targetLocale === 'en') return obj;
    const clone = deepClonePreservingDates(obj);
    const extracted: ExtractedField[] = [];
    for (const path of fieldPaths) {
        walkPath(clone, path.split('.'), (parentNode, key) => {
            const value = parentNode[key as keyof typeof parentNode];
            if (typeof value === 'string' && value.length > 0) {
                extracted.push({ get: () => ..., set: (translated: string) => { ... } });
            }
        });
    }
    if (extracted.length === 0) return clone;
    try {
        const translated = await this.translateStrings(extracted.map((e) => e.get()), targetLocale);
        extracted.forEach((entry, i) => entry.set(translated[i]));
        return clone;
    } catch (err) {
        this.logger.error(`Translation to locale "${targetLocale}" failed, returning original content untranslated`, err as Error);
        return obj;
    }
}
```

The caller (in this cluster, the TLD content service covered in [02-static-content-and-tld-metadata.md](02-static-content-and-tld-metadata.md)) does not hand this function a whole entity and hope for the best, it hands it a specific list of dot paths, things like `content.domainHero.headline` or `faqs[].question`, and only the string values sitting at exactly those paths get sent to Azure and swapped back in. Everything else in the object, ids, urls, booleans, the stat value that happens to look like text but is really data, is left completely untouched, because the walker never visits it. `deepClonePreservingDates` exists so this process never mutates the original database record in place, and so that any `Date` instances inside the object survive the clone as real `Date` objects rather than getting turned into strings, which a naive `JSON.parse(JSON.stringify(...))` clone would have done.

The failure handling is just as deliberate, if the Azure call throws for any reason, the whole function logs the error and returns the original, untranslated object rather than letting a translation vendor outage take down the page entirely. A user asking for Arabic gets the English version rather than an error page if Azure is briefly unavailable.

## Caching that invalidates itself without anyone writing an invalidation step

`LocalizedTranslationCacheService` is a thin wrapper around the exact same global `CACHE_MANAGER` that `app.module.ts` registers for the whole application (the same cache backing `CacheInterceptor` elsewhere in this cluster), just used programmatically here instead of through a decorator. The interesting part is the cache key itself:

```ts
export function buildTranslationCacheKey(resource: string, recordId: string, locale: string, updatedAt: Date | string | null | undefined): string {
    const updatedAtEpoch = updatedAt ? new Date(updatedAt).getTime() : 0;
    return `translate:v1:${resource}:${recordId}:${locale}:${updatedAtEpoch}`;
}
```

The record's own `lastChangedDateTime` timestamp is baked directly into the cache key. When an admin edits a TLD's content, that timestamp changes, which means the next request for a translation of that record computes an entirely different cache key and simply misses, forcing a fresh translation. Nothing anywhere in this codebase has to remember to call an `invalidate` method when content changes, the old cache entry does not get deleted, it just becomes unreachable and ages out on its own seven day TTL. This is a genuinely elegant way to sidestep cache invalidation bugs, worth remembering as a pattern the next time a caching problem looks like it needs an explicit invalidation hook.

## Where the credentials come from, and why they are simpler than Google's

`AzureTranslatorConfigService` pulls `AZURE_TRANSLATOR_KEY`, `AZURE_TRANSLATOR_REGION`, `AZURE_TRANSLATOR_ENDPOINT`, and `AZURE_TRANSLATOR_API_VERSION` out of the same shared AWS Secrets Manager bundle every other service in this codebase reads from (see [03-configuration-and-secrets.md](../../03-configuration-and-secrets.md) for how that mechanism works generally), caches them in memory after the first successful fetch, and throws a clear error naming exactly which keys are missing if any are absent. A comment on the class is worth quoting directly, because it tells you something true about this vendor choice compared to the Google integrations covered later in this cluster: "Azure Translator authenticates with a static subscription key plus a region header, no OAuth or JWT flow, so this is deliberately much simpler than the Google service account auth it replaces." That is a real, useful contrast, Google's analytics and search console integrations (files 06 and 07) both require minting and caching a signed JWT client from a service account private key, while this one only ever needs two header values.
