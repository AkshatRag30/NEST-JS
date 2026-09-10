# 03. Builder Profile, a Link-in-Bio Page for a .og Domain

## Confirming the hypothesis, from the actual routes

The idea going in was that `builder-profile` sounds like a public-facing profile page tied to a user's domain, something like a personal link-in-bio built on a Web3 domain. Reading the controller confirms this almost exactly:

```ts
// src/components/builder-profile/controllers/builder-profile.controller.ts
@ApiTags('Builder Profile')
@Controller('profile')
export class BuilderProfileController {
    @Post()
    @UseGuards(AccessTokenGuard, OgDomainGuard)
    async createProfile(@Req() req: Request, @Body() dto: CreateProfileDto) { ... }

    @Put()
    @UseGuards(AccessTokenGuard, OgDomainGuard)
    async updateProfile(@Req() req: Request, @Body() dto: UpdateProfileDto) { ... }

    @Post('avatar')
    @UseGuards(AccessTokenGuard, OgDomainGuard)
    @UseInterceptors(FileInterceptor('avatar'))
    async uploadAvatar(...) { ... }

    @Post('project')
    @UseGuards(AccessTokenGuard, OgDomainGuard)
    async addProject(...) { ... }

    @Put('socials')
    @UseGuards(AccessTokenGuard, OgDomainGuard)
    async upsertSocials(...) { ... }

    @Get(':domain')
    @UseGuards(OptionalAuthGuard)
    async getPublicProfile(@Req() req: Request, @Param('domain') domainName: string) { ... }
}
```

Every write route requires `OgDomainGuard` on top of the normal `AccessTokenGuard`, and that guard (`src/components/reputation-gm-perk/guards/og-domain.guard.ts`) does exactly one thing, count how many `.og` domains the caller owns, and throw a 403 if that count is zero. So this feature is deliberately gated to people who own at least one `.og` domain, it isn't a general profile page available to every account, it's a perk tied to owning a specific product this company sells. And the read side, `GET /profile/:domain`, takes a domain name directly in the URL, `guru.og`, resolves it to whichever profile is bound to it, and serves it back, which is precisely the "visit someone's domain, see their page" shape of a link-in-bio product.

## The entity, and the one-profile-per-domain rule

```ts
// src/components/builder-profile/entities/builder-profile.entity.ts
@Entity({ name: 'tbl_builder_profile' })
@Index('idx_builder_profile_user', ['userId'])
export class BuilderProfileEntity extends BaseEntity {
    @Column({ type: 'uuid', nullable: false })
    public userId: string;

    @Column({ unique: true, nullable: true })
    public primaryDomainId: string | null;

    @Column({ nullable: true })
    public username: string;

    @Column({ type: 'text', nullable: true })
    public bio: string;

    @Column({ nullable: true })
    public avatarUrl: string;

    @Column({ type: 'jsonb', default: () => "'[]'" })
    public skills: string[];

    @Column({ type: 'boolean', default: false })
    public isPublished: boolean;
    ...
```

The unique constraint sits on `primaryDomainId`, not on `userId`, which is exactly why a user can only have one active profile at a time, but it's bound to their domain, not to their account. A comment right above the column explains why it's nullable at all:

```ts
// Nullable so a prior owner's row can be soft-unbound (rather than deleted) once
// a new owner opts in on the same domain. Postgres never treats two NULLs as equal,
// so the unique constraint still allows any number of unbound rows.
```

That's the same `NULL`-in-a-unique-column trick seen on `User.email` and `WalletAddressEntity.walletAddress`, reused here for a different reason, once a `.og` domain gets resold, the previous owner's profile row survives (nothing gets deleted), it just gets its `primaryDomainId` cleared to `null` and `isPublished` flipped to `false`, so the domain is free for the new owner to bind a fresh profile to. `skills` is a `jsonb` array with a raw SQL default (`'[]'`), a plain string array validated at the application layer, not by a Postgres check constraint or an enum type.

`BuilderProjectEntity` and `BuilderSocialEntity` hang off this same profile row. Projects are a genuine one-to-many, `profileId` has no unique constraint, so a builder can add as many project cards as they like. Socials is the opposite, `profileId` is `unique` on `BuilderSocialEntity`, meaning each profile has at most one socials row that gets upserted in place rather than a growing list.

## Creating a profile, and what "primary domain" really means here

```ts
// src/components/builder-profile/services/builder-profile.service.ts
async createProfile(userId: string, dto: CreateProfileDto): Promise<BuilderProfileEntity> {
    this.assertValidSkills(dto.skills);

    const user = await this.userRepo.findById(userId);
    if (!user?.primaryDomainId) {
        throw new BadRequestException('No primary domain set for this user.');
    }

    const domain = await this.domainBcRepo.findByTokenIdAndUserId(user.primaryDomainId, userId);
    if (!domain) {
        throw new ForbiddenException('The provided primaryDomainId does not belong to your account.');
    }

    const existing = await this.builderProfileRepo.findByPrimaryDomainId(user.primaryDomainId);
    if (existing) {
        if (existing.userId === userId) {
            throw new ConflictException('A builder profile already exists for your primary domain.');
        }
        await this.builderProfileRepo.update(existing.id, { primaryDomainId: null, isPublished: false });
    }

    return this.builderProfileRepo.create({ userId, primaryDomainId: user.primaryDomainId, ...dto });
}
```

Notice this reads `user.primaryDomainId` off `tbl_user` itself, the column covered in the user note, it doesn't take a domain id from the request body at all. So creating a profile is really a two-step process from a user's point of view: first set a `.og` domain as your primary one (elsewhere in the app), then call this endpoint, which binds a fresh profile to whatever that pointer currently says. The `domainBcRepo.findByTokenIdAndUserId` call is a second, independent check that the caller actually still owns that domain right now, not just that the pointer exists, `OgDomainGuard` already proved the caller owns at least one `.og` domain, this proves it's specifically the one their `primaryDomainId` points to.

The soft-unbind branch here is exactly the resale scenario the entity comment describes, and it's directly covered by a test:

```ts
// src/components/builder-profile/services/builder-profile.service.spec.ts
it("soft-unbinds a prior owner's existing row (primaryDomainId: null, isPublished: false) instead of deleting it or reassigning it, when a new owner creates on the same domain", async () => {
    ...
    await service.createProfile('user-1', { username: 'new.owner' });

    expect(builderProfileRepo.update).toHaveBeenCalledWith('prior-owner-profile', { primaryDomainId: null, isPublished: false });
    expect(builderProfileRepo.create).toHaveBeenCalledWith({ userId: 'user-1', primaryDomainId: 'domain-1', username: 'new.owner' });
});
```

## Skills validation happens in the service, not the DTO

```ts
// src/components/builder-profile/dtos/create-profile.dto.ts
@IsOptional()
@IsArray()
@IsString({ each: true })
skills?: string[];
```

`class-validator` on the DTO only confirms `skills` is an array of strings, it says nothing about which strings are allowed. The actual allow-list lives in a small constants file:

```ts
// src/components/builder-profile/constants/builder-profile-skills.constant.ts
export const BUILDER_PROFILE_SKILLS = ['Solidity', 'Rust', 'TypeScript', 'Design', 'Community', 'Marketing'] as const;
```

and gets enforced by hand inside the service, on both create and update:

```ts
private assertValidSkills(skills?: string[]): void {
    if (!skills?.length) return;
    const invalid = skills.filter((skill) => !BUILDER_PROFILE_SKILLS.includes(skill as (typeof BUILDER_PROFILE_SKILLS)[number]));
    if (invalid.length) {
        throw new BadRequestException(
            `Invalid skill: '${invalid[0]}'. Allowed skills: ${BUILDER_PROFILE_SKILLS.join(', ')}.`
        );
    }
}
```

The controller also exposes this exact same list as public data, `GET /profile/skills`, specifically so a frontend can render the real, current set of checkboxes instead of hardcoding a copy of it that could drift out of sync:

```ts
// NOTE: must stay registered before any GET '/:domain' style route (Sprint 11), or Nest will try to resolve 'skills' as a domain param.
@Get('skills')
getSkillsList() {
    return { data: BUILDER_PROFILE_SKILLS };
}
```

That comment is worth reading closely, it's a real, in-code warning left by whoever wrote this, about route ordering. Nest matches routes in registration order, and `@Get(':domain')` would happily swallow a request to `/profile/skills` by treating `"skills"` as a domain name, if it were registered above this one. The exact same warning, mirrored, sits above the `:domain` route itself further down the file. This is a genuinely useful thing to internalize before writing your own controllers, a parameterized route and a static one that could collide always need the static one registered first.

## Avatar upload, S3, and what the DTO layer cannot validate

```ts
// src/components/builder-profile/constants/builder-profile.constants.ts
export const ALLOWED_AVATAR_MIME_TYPES = ['image/jpeg', 'image/png', 'image/webp'];
export const MAX_AVATAR_SIZE_BYTES = 5 * 1024 * 1024;
```

There is no DTO for the avatar upload route at all, `@UploadedFile()` hands the service a raw `Express.Multer.File`, and mime type and size are both checked by hand at the top of `uploadAvatar`:

```ts
async uploadAvatar(userId: string, file: Express.Multer.File): Promise<{ avatarUrl: string }> {
    if (!ALLOWED_AVATAR_MIME_TYPES.includes(file?.mimetype)) {
        throw new BadRequestException('Avatar must be a JPEG, PNG, or WEBP image under 5MB.');
    }
    if (file.size > MAX_AVATAR_SIZE_BYTES) {
        throw new BadRequestException('Avatar must be a JPEG, PNG, or WEBP image under 5MB.');
    }

    const profile = await this.getOwnProfileOrThrow(userId);

    const dir = `builder-profile/avatars/${userId}-`;
    const key = await this.s3Service.uploadFile(file, dir, this.S3_BUCKET_NAME, 'public-read');
    const encodedKey = key.split('/').map(encodeURIComponent).join('/');
    const avatarUrl = `https://${this.S3_BUCKET_NAME}.s3.amazonaws.com/${encodedKey}`;

    await this.builderProfileRepo.update(profile.id, { avatarUrl });
    return { avatarUrl };
}
```

`file.mimetype` is exactly the kind of value `class-validator` decorators can't reach, since it lives on a Multer-parsed multipart upload, not the JSON body a `ValidationPipe` inspects, so this check has to be written out explicitly rather than declared. Two things worth noticing for a frontend developer used to client-side "only accept .png/.jpg" file input hints: the mime type here comes from what the browser or client reports in the multipart request, not from actually inspecting the file's bytes, and the bucket name itself (`S3_IMG_BUCKET_NAME`) is read once, in the constructor, off `ConfigService`, which ultimately traces back to the AWS Secrets Manager bundle described in this codebase's configuration note, not a hardcoded string anywhere in this file.

`getOwnProfileOrThrow` is called after the file validation, and it's the same ownership gate that guards every other own-profile write, covered in full in the next note.

## The public profile, and everything it borrows from other clusters

```ts
async getPublicProfile(domainName: string, callerUserId: string | null) {
    const { profile, isOwner } = await this.resolvePublishedProfileOrThrow(domainName, callerUserId);

    const [projects, socials, reputation, gmStreak, nftCollections, contractDeployments] = await Promise.all([
        this.builderProjectRepo.findByProfileId(profile.id),
        this.builderSocialRepo.findByProfileId(profile.id),
        this.getReputationSummary(profile.primaryDomainId),
        this.gmService.getMyStreak(profile.userId, profile.primaryDomainId).catch(() => this.emptyGmStreak()),
        this.nftCollectionRepo.findConfirmedByPrimaryDomainId(profile.primaryDomainId).catch(() => []),
        this.contractDeploymentRepo.findConfirmedByPrimaryDomainId(profile.primaryDomainId).catch(() => []),
    ]);

    return {
        ...profile,
        domainName,
        isOwner,
        projects,
        socials: socials ?? this.emptySocials(),
        reputation,
        gmStreak,
        achievements: {
            nftCollections: nftCollections.map((c) => ({ id: c.id, name: c.name, contractAddress: c.contractAddress, confirmedAt: c.lastChangedDateTime })),
            contractDeployments: contractDeployments.map((d) => ({ id: d.id, name: d.configSnapshot?.name ?? null, contractAddress: d.contractAddress, confirmedAt: d.lastChangedDateTime })),
        },
    };
}
```

This one endpoint is genuinely the richest single method in this whole cluster, it's a public "profile page" payload assembled live from five different services on every request, projects and socials from this module's own repositories, a reputation score and tier from the reputation cluster, a GM check-in streak from the GM/perk cluster, and confirmed NFT collections and contract deployments from two entirely separate clusters, all keyed off the same `primaryDomainId`. Every one of the cross-cluster calls (reputation, GM streak, NFTs, contracts) is individually wrapped in its own `.catch()`, falling back to `null` or an empty array rather than letting one flaky dependency 500 the whole profile page, and a dedicated test confirms exactly that, `nftCollectionRepo` rejecting still leaves `reputation` and `gmStreak` populated normally. This is a genuinely good, reusable pattern, when a page aggregates several independent data sources, one of them failing should degrade that one section, not the whole response.

One more thing worth knowing if you ever end up debugging this feature, a test in this same spec file asserts, by reading the actual source text of `builder-profile.service.ts`, that it never imports `calculateTotalScore`, the write-side reputation scorer:

```ts
it('never imports or calls calculateTotalScore, the write-side scorer, from this read-path service', () => {
    const source = readFileSync(join(__dirname, 'builder-profile.service.ts'), 'utf8');
    expect(source).not.toMatch(/calculateTotalScore/);
});
```

That's an unusual kind of test, a static grep against the file's own text rather than a behavioral assertion, and it exists specifically to keep a public, read-only, potentially anonymous GET route from ever accidentally triggering an expensive or state-mutating recalculation as a side effect of someone just viewing a page.
