# 03. Domain Provider and TLDs, the Table That Makes the Whole Design Work

## A small table carrying a lot of weight

`domain-provider-and-tlds` is one of the smallest sub modules in this folder, a single entity, a thin service, and a repository, but almost everything else in this module leans on it. Its entire job is answering one question, given a top level domain like `.crypto`, `.eth`, or `.bnb`, which of this company's supported providers actually issues it.

```ts
// src/components/domain/domain-provider-and-tlds/entity/domain-provider-and-tlds.entity.ts
@Entity({ name: 'tbl_domain_provider_and_tlds' })
export class DomainProviderAndTldsEntity extends BaseEntity {
    @Column({ unique: true, nullable: false }) domain_provider_and_tlds_unique_identifier: string;
    @Column({ nullable: false }) domain_provider: string;
    @Column({ nullable: false }) abbreviation: string;
    @Column({ nullable: false }) tld: string;
}
```

Four columns, and the `04-database-typeorm-and-repositories.md` note already flagged that this exact table gets its baseline data from a real seed file, `create-domain-provider-and-tld.seed.ts`, run once through the `typeorm-extension` seeding CLI, populating which providers and which TLDs exist before any real user activity happens against the system.

## The service is a thin pass through, on purpose

```ts
// src/components/domain/domain-provider-and-tlds/domain-provider-and-tlds.service.ts
async getAllDomainProviderAndTld(): Promise<DomainProviderAndTldsEntity[]> {
    return this.domainProviderAndTldsRepoInterface.findAll();
}
async getByDomainProviderAndTld(domainProvider: string, tld: string): Promise<DomainProviderAndTldsEntity> {
    return this.domainProviderAndTldsRepoInterface.findByDomainProviderAndTld(domainProvider, tld);
}
async getByTld(tld: string): Promise<DomainProviderAndTldsEntity> {
    return this.domainProviderAndTldsRepoInterface.findByTld(tld);
}
```

Every method here is a near direct forward to the repository, with no business logic of its own, which is exactly right for what this module is, a lookup, not a workflow. The one method that does carry real logic is `getAllDomainProviderAndTldForAdmin`, which groups every row by provider and explicitly excludes a hardcoded list of providers (`Tezos`, `Aptos`, `Ton`, `Avax`, `Starknet`, `Box`) from whatever admin facing list consumes it, worth remembering as one of the small signs throughout this codebase that not every provider this company has code for is necessarily fully live or fully supported in every part of the product at the same time.

## Where this table actually gets used

`getByTld` specifically is the method `DomainSearchService.checkDomainProvider` calls (see the previous note) on every single search that includes a TLD, wrapped in its own ten minute cache precisely because this table changes so rarely that querying it on every keystroke would be wasted work. Anywhere else in this module that needs to translate a bare TLD string into "which of our eleven possible provider integrations do I actually call" ultimately traces back to this same small table.
