# 04. The Ownership Staleness Gate, and the Admin Side of Builder Profile

## Why "who owns this domain" needs its own service

A `.og` domain is an NFT. NFTs get sold. `builder-profile` is built entirely on top of `tbl_user.primaryDomainId`, a plain column that only changes when a user explicitly re-points it, so it is a manually-set pointer, not proof of anything happening right now. Nothing stops a user from selling their domain on a marketplace elsewhere in this same codebase without ever touching that pointer. If nothing checked for that, a sold domain's old profile page would keep resolving publicly under the previous owner's content indefinitely, and the previous owner could keep editing a profile bound to a domain that now belongs to someone else entirely.

`BuilderProfileOwnershipService` exists specifically to close that gap:

```ts
// src/components/builder-profile/services/builder-profile-ownership.service.ts
const OWNERSHIP_CACHE_TTL_MS = 60_000;

@Injectable()
export class BuilderProfileOwnershipService {
    private readonly ownershipCache = new Map<string, CachedOwnership>();

    async isCurrentlyOwned(userId: string, primaryDomainId: string): Promise<boolean> {
        const cached = this.ownershipCache.get(primaryDomainId);
        if (cached && Date.now() - cached.checkedAt < OWNERSHIP_CACHE_TTL_MS) {
            return cached.owned;
        }

        const domain = await this.domainBcRepo.findByTokenIdAndUserId(primaryDomainId, userId);
        const owned = !!domain;
        this.ownershipCache.set(primaryDomainId, { owned, checkedAt: Date.now() });
        return owned;
    }
}
```

It asks the domain-detail blockchain repository (owned by a different cluster) whether this specific `userId` currently owns this specific `primaryDomainId`, and caches the answer, positive or negative, for sixty seconds, keyed by domain id. The comment on the constant spells out the tradeoff directly: `// How long a positive/negative ownership result may be served from cache before re-checking the domain repo. Bounds how long a transfer can stay invisible.` This is a deliberately small, explicit staleness window, not a correctness guarantee, a transfer can still be invisible to this check for up to sixty seconds, and the code says so rather than pretending otherwise.

This is called from two different places for two different reasons.

## Guarding every owner write

```ts
// src/components/builder-profile/services/builder-profile.service.ts
private async getOwnProfileOrThrow(userId: string): Promise<BuilderProfileEntity> {
    const user = await this.userRepo.findById(userId);
    if (!user?.primaryDomainId) {
        throw new BadRequestException('No primary domain set for this user.');
    }

    const profile = await this.builderProfileRepo.findByPrimaryDomainId(user.primaryDomainId);
    if (!profile || profile.userId !== userId) {
        throw new NotFoundException('Builder profile not found. Create one first.');
    }

    // tbl_user.primaryDomainId is a manually-set pointer, not proof of current
    // ownership — it only changes when the user explicitly sets a new primary
    // domain. If the domain was transferred away without that happening, block
    // edits rather than silently saving to an orphaned profile.
    const stillOwned = await this.ownershipService.isCurrentlyOwned(profile.userId, profile.primaryDomainId as string);
    if (!stillOwned) {
        throw new ConflictException('Cannot edit a profile whose domain you no longer own. Update your primary domain first.');
    }

    return profile;
}
```

Every own-profile mutation, `updateOwnProfile`, `uploadAvatar`, `addProject`, `removeProject`, `upsertSocials`, routes through this one helper first. It's a three-part check every time: does this user have a `primaryDomainId` set at all, does a profile row exist for that domain and belong to this exact user, and, only then, does the ownership service confirm that domain is still actually theirs right now. Fail any one of those and the write never happens. A dedicated describe block in the spec file walks through exactly this scenario:

```ts
// src/components/builder-profile/services/builder-profile.service.spec.ts
describe('BuilderProfileService — domain transfer staleness gate on owner writes', () => {
    it('rejects updateOwnProfile with a 409 ConflictException, not a silent save, when the domain is no longer owned', async () => {
        ...
        await expect(service.updateOwnProfile('user-1', { bio: 'edit after transfer' })).rejects.toBeInstanceOf(ConflictException);
        expect(builderProfileRepo.update).not.toHaveBeenCalled();
    });
```

A 409 Conflict, not a 403 or a 404, which is the right HTTP semantics here, the resource and the request are both fine, it's the caller's relationship to that resource that has changed underneath them since they last checked.

## Guarding every public read

The exact same problem exists on the read side, just phrased differently: a domain that's been sold shouldn't keep serving its old owner's published profile to strangers just because the row is still marked `isPublished: true`.

```ts
private async resolvePublishedProfileOrThrow(domainName: string, callerUserId: string | null) {
    const domain = await this.domainBcRepo.findByDomainName(domainName);
    if (!domain) throw new NotFoundException('Builder profile not found.');

    const profile = await this.builderProfileRepo.findByPrimaryDomainId(domain.token_id);
    if (!profile) throw new NotFoundException('Builder profile not found.');

    const isOwner = callerUserId === profile.userId;

    // A domain transfer must stop a profile from resolving publicly the moment it
    // happens, without waiting on a background sweep. A caller who isn't the profile's
    // original creator gets the same 404 as a profile that never existed.
    const stillOwned = await this.ownershipService.isCurrentlyOwned(profile.userId, profile.primaryDomainId as string);
    if (!stillOwned && !isOwner) {
        throw new NotFoundException('Builder profile not found.');
    }

    if (!profile.isPublished && !isOwner) {
        throw new NotFoundException('Builder profile not found.');
    }

    return { profile, isOwner };
}
```

Two things stand out here. First, this deliberately returns the same 404 for "never existed," "domain transferred away," and "unpublished draft viewed by a stranger", it never leaks the distinction between those three cases to an anonymous caller, which is a small but real information-hiding choice. Second, notice the `isOwner` check is checked before the ownership-service call is even used to gate anything, `isOwner` alone doesn't bypass the transfer check, look again: `if (!stillOwned && !isOwner)` still blocks a non-owner even if the row claims to be published. Only the profile's own original creator can still see it once the domain is gone, everyone else gets treated exactly as if the profile never existed. This is directly tested:

```ts
it('throws NotFoundException for a non-owner when the domain has been transferred away, even though the profile row still exists and is published', async () => {
    ...
    await expect(service.getPublicProfile('guru.og', 'someone-else')).rejects.toBeInstanceOf(NotFoundException);
    await expect(service.getPublicProfile('guru.og', null)).rejects.toBeInstanceOf(NotFoundException);
});
```

Worth noticing as a frontend developer specifically: this means a "profile not found" 404 from this endpoint is genuinely ambiguous by design, and a frontend can't distinguish "this domain never had a profile" from "this domain used to have one, it just got sold" from the response alone. That's a deliberate privacy tradeoff on the backend's part, not a gap in the API contract.

## The admin side, a separate controller entirely

`AdminBuilderProfileController` lives in the same file as the public controller but is a fully separate `@Controller('admin/builder-profiles')`, gated by `AccessTokenGuard` plus `SuperAdminAccessGuard` on every route, not the `.og`-domain-ownership guard the public controller uses. This is a straightforward internal tooling surface: list all profiles with filters, get stats, view one profile's full detail, and archive or restore a profile.

```ts
// src/components/builder-profile/repositories/builder-profile.repository.ts
async findAllForAdmin(pagination, filters: AdminBuilderProfileFilters = {}) {
    ...
    if (search?.trim()) {
        const s = search.trim();
        qb = qb.andWhere(
            new Brackets((inner) => {
                inner
                    .where('LOWER(profile.username) LIKE LOWER(:search)', { search: `%${s}%` })
                    .orWhere('LOWER(profile.bio) LIKE LOWER(:search)', { search: `%${s}%` })
                    .orWhere(`profile."userId"::text IN (SELECT u.id::text FROM tbl_user u WHERE LOWER(u.email) LIKE LOWER(:emailSearch))`, { emailSearch: `%${s}%` })
                    .orWhere(`profile."primaryDomainId" IN (SELECT d.token_id FROM tbl_domain_detail_bc d WHERE LOWER(d."domainName") LIKE LOWER(:domainSearch))`, { domainSearch: `%${s}%` });
            })
        );
    }
    ...
}
```

One search box on the admin list actually checks four different things at once, username, bio, the owning user's email (via a subquery against `tbl_user`), and the bound domain's name (via a subquery against `tbl_domain_detail_bc`), because none of those last two live directly on `BuilderProfileEntity` itself. That's why `findAllForAdmin` and `findByIdForAdmin` both call a private `enrichWithUserEmailAndDomainName` helper afterward, a raw SQL query that batches up every distinct `userId` and `primaryDomainId` in the current page of results and joins the email and domain name back on in memory, rather than doing it as one giant SQL join, presumably because those two tables belong to different modules entirely and TypeORM's relations aren't wired across them here.

"Archive" and "restore" are soft-delete toggles, not real deletes:

```ts
async softDeleteById(id: string): Promise<void> {
    await this.builderProfileRepo.update(id, { isDeleted: true });
}
async restoreById(id: string): Promise<void> {
    await this.builderProfileRepo.update(id, { isDeleted: false });
}
```

and every admin listing query explicitly filters `WHERE profile.isDeleted = false` so an archived profile simply stops showing up, its row and its projects and socials all stay in the database untouched, ready to be restored by flipping the same flag back.
