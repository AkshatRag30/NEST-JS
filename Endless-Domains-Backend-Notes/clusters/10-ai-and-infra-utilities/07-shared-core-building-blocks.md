# 07. The shared building blocks: base entity, response shapes, enums, and validation

## `BaseEntity`, the class almost every table in this application extends

`src/@core/common/entity/base.entity.ts` is four fields long, and it is, by a wide margin, the single most depended upon piece of shared code covered anywhere in this cluster.

```ts
export abstract class BaseEntity {
    @PrimaryGeneratedColumn('uuid')
    id: string;

    @CreateDateColumn({ type: 'timestamptz', default: () => 'CURRENT_TIMESTAMP' })
    createdDateTime: Date;

    @Column({ type: 'boolean', default: false })
    isDeleted: boolean;

    @UpdateDateColumn({ type: 'timestamptz', default: () => 'CURRENT_TIMESTAMP' })
    lastChangedDateTime: Date;
}
```

A quick search across the whole `src` tree for `extends BaseEntity` turns up ninety nine separate entity files. Compare that to `retryWithBackoff` from note 06, used in a small handful of places, or `MailService`, called from three, and the difference in scale is the whole point, `BaseEntity` is not one utility among many, it is the foundation almost the entire data model sits on. Every table that extends it automatically gets a UUID primary key rather than an auto incrementing integer (a deliberate, common choice in systems where IDs might need to be generated client side or merged across databases without collision), a `createdDateTime` and `lastChangedDateTime` that TypeORM manages automatically without any feature code ever setting them by hand, and, most importantly, `isDeleted`, a boolean that is what makes soft deletes possible everywhere in this app. A row with `isDeleted: true` is not actually gone from the database, note 03's `AiDomainAdvisorService.toolGetMyDomains` filtering with `.andWhere('d.isDeleted = false')` is a direct, visible example of a feature module having to respect that convention rather than trusting a plain SQL `DELETE` to have removed a row for good.

Every AI entity covered earlier in this cluster, `AiDomainDiscoveryLogEntity`, `AiConversationLogEntity`, `DomainAdvisorLogEntity`, `AiSubscriptionIntentEntity`, extends this exact class, which is why none of them had to define their own `id`, timestamp, or soft delete columns by hand, and it is a strong, concrete answer to the question of which shared `@core` utility every other cluster's code most depends on.

## `Response` and `ResultResponse`, the two shapes every controller answers with

`src/@core/common/dto/response.dto.ts` is the wrapper almost every controller in this codebase returns from almost every endpoint, and it is genuinely simple.

```ts
export class Response {
    success: boolean;
    statusCode: number;
    message: string[] | string;
    result: any;
    constructor(message: string[] | string, result?: any) { this.message = message; this.result = result; }
}
```

Every controller method throughout this whole cluster, `new Response('Domain name candidates generated', result)` in note 01, `new Response('Subscription activated successfully', { activated: true })` in note 02, follows this same shape, a human readable message plus whatever the actual payload is. This is what gives a frontend developer a single, predictable envelope to unwrap on every API call in the app, rather than each endpoint inventing its own response shape.

`ResultResponse` (`result-response.dto.ts`) is the more specific cousin used for paginated list endpoints specifically, wrapping raw `data` alongside an optional `paginationInfo` block, and it is what `paginateResponse` in `helper.ts` (covered in note 06's companion file) actually constructs. `PaginationInfo` (`pagination-info.dto.ts`) is the plain shape that block takes, `count`, `currentPage`, `nextPage`, `prevPage`, and `lastPage`, and `PaginationRequestDto` (`pagination-request.dto.ts`) is the equivalent shape going the other direction, what a paginated request looks like once it has been parsed, `limit`, `skip`, `page`, an optional `sort`, `search`, and `tlds` filter. `ProviderManagementUserRepo` from note 04 is a direct, visible user of exactly this pairing, `getListOfAllDomainProvider` builds a `PaginationRequestDto` from the incoming query and hands it straight to `paginateResponse` to build the final response shape.

`DateRequestDto` and `DateFilterRequestDto` (`date-request.dto.ts`, `date-filter-request.dto.ts`) are the equivalent shared shapes for any endpoint that accepts an optional date range filter, and `DateRequestDto` is where the one custom validation decorator in this cluster's scope actually gets applied.

## The one pair of custom validation decorators in scope

`src/@core/common/custom-validation-decorator/fromdate-todate-validation-decorator.ts` defines two decorators, `FromDateExistsIfToDateExists` and `ToDateExistsIfFromDateExists`, built with `class-validator`'s own `registerDecorator` escape hatch for writing a validation rule that does not exist as a built in decorator. The rule they enforce, together, is that a date range filter has to be given as a genuine pair, if a caller supplies `fromDate` but no `toDate`, or the reverse, the request should be rejected rather than silently treated as an open ended range, and if both are present, `toDate` actually has to fall on or after `fromDate`.

```ts
validate(value: any, args: ValidationArguments) {
    const fromDate = (args.object as Record<string, string>)['fromDate'];
    const toDate = value;
    if (fromDate && !toDate) { return false; }
    if (fromDate && toDate && new Date(toDate) < new Date(fromDate)) { return false; }
    return true;
}
```

The reason this needed a custom decorator at all, rather than something built into `class-validator` out of the box, is that the rule depends on two different fields on the same object at once, whether `toDate` is valid depends on whether `fromDate` was also given, which is exactly the kind of cross field validation the library's simpler built in decorators (`@IsString`, `@IsOptional`, and so on) cannot express by themselves. `DateRequestDto` applies both decorators, one to each field, so that either field validates itself against whatever the other field's current value is.

## `EnumValidationPipe`, a pipe for validating a route parameter against an arbitrary enum

`src/@core/validation-pipes/enum-validation-pipe.ts` is a small, generic `PipeTransform` that any controller can use to make sure a raw string coming in through a query parameter or route parameter is actually one of the values a given TypeScript enum defines, throwing a clear `BadRequestException` naming every acceptable value if it is not.

```ts
export class EnumValidationPipe implements PipeTransform<string, Promise<any>> {
    constructor(private enumEntity: any) {}
    transform(value: string): any {
        if (isDefined(value) && isEnum(value, this.enumEntity)) { return value; }
        const errorMessage = `the value ${value} is not valid. See the acceptable values: ${Object.keys(this.enumEntity).map((key) => this.enumEntity[key])}`;
        throw new BadRequestException(errorMessage);
    }
}
```

Because its constructor takes the enum itself as an argument, one single pipe class can validate against any enum in the app, `new EnumValidationPipe(DomainProvider)` in one controller, `new EnumValidationPipe(SortEnum)` in another, without needing a separate pipe written per enum. This is the same instinct behind `BaseAbstractRepository` from note 04 of the wider notes set, one small, generic, reusable class standing in for what could otherwise have been dozens of nearly identical, purpose built ones.

## The shared enums, and what they tell you about the business without reading a single service

`src/@core/common/enum/` and `src/@core/enum/` together hold a long list of small, plain enums, and reading through them in order is a genuinely fast way to absorb real facts about the business without opening a single service file.

`DomainProvider` lists every registrar and blockchain naming service this platform actually integrates with, `UNSTOPPABLE_DOMAINS`, `ENS`, `Freename`, `Bonfida`, `Tezos`, `Arbitrum`, `BinanceSmartChain`, `Aptos`, `Ton`, `Avax`, a Base flavored variant of Unstoppable Domains, Solana, Starknet, and Box, a longer list than the ten providers `AiDomainAdvisorService`'s own `TLD_KNOWLEDGE` map in note 02 actually has detailed knowledge entries for, which tells you the Advisor's knowledge base has not been kept perfectly in sync with every provider the platform has gone on to support since. `Blockchain`, `BlockchainNetwork`, `BlockchainEnvironment`, and `BlockchainExplorer` describe the underlying chains, whether an environment is a mainnet or a testnet, and which block explorer URL to link to for a given chain and network combination, useful context for understanding why an on chain transaction confirmation email might need to build a different explorer link depending on which chain the domain in question actually lives on.

`AdminRole` (`SUPER_ADMIN`, `ADMIN`, `MARKETING`) is the plain string enum backing the role checks `SuperAdminAccessGuard` and friends perform, and its own comment is a useful, honest note about consistency, "`SuperAdminAccessGuard` checks its two roles as inline literals and is left untouched; new guards ... should reference this enum instead of introducing more hardcoded literals," an explicit acknowledgment that an older guard still hardcodes its role strings rather than importing this enum, while newer guards are expected to do better. `DomainOrderStatus` and `MintStatus` describe the lifecycle states an order or an on chain mint can be in, `Pending`, `Processing`, `Completed`, `Cancelled` or `Failed`, `On_Hold`, which is directly useful context for reading `TasksService`'s domain expiry and mint status jobs from note 05. `CompanyInfo` holds the platform's own registered legal name and address, "Endless Domains Ltd." and its Hong Kong registered address, presumably used somewhere a legal footer needs to print the company's real registered details rather than its consumer facing brand name.

`domainProviderList` (`static-data/domain-provider-list.ts`) is a small, plain array pairing four of the `DomainProvider` enum values together, `Bonfida`, Unstoppable Domains, ENS, and Tezos, worth noticing as an incomplete looking subset of the full `DomainProvider` enum rather than every provider the platform actually supports, presumably a seed list or default set for some narrower purpose rather than the canonical list of everything the platform integrates with.
