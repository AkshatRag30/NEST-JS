# 02. Wallet Address and the Web3 Linkage Point

## A module with no controller

`src/components/wallet-address` has an entity, a repository, an interface, and a module, and nothing else. No controller. Nothing in this folder answers an HTTP request directly. `WalletAddressModule` exports its repository interface and nothing more:

```ts
// src/components/wallet-address/wallet-address-module.ts
@Module({
    imports: [TypeOrmModule.forFeature([WalletAddressEntity]), LoggerModule],
    providers: [{ provide: 'WalletRepoInterface', useClass: WalletRepo }],
    exports: [{ provide: 'WalletRepoInterface', useClass: WalletRepo }]
})
export class WalletAddressModule { }
```

That's the whole shape of it, a pure data layer that other modules reach into. `UserModule` imports it directly, and `UserService` is the actual consumer, every wallet-related method a client ever calls (linking a wallet, checking a nonce, resolving a user by wallet address) is a `users` route or an auth route that delegates down into `WalletRepoInterface`. This is also the seam this note was asked to name explicitly: the Web3 auth cluster's wallet-signature login flow, and the blockchain integration cluster's on-chain lookups, both ultimately read and write the exact same `tbl_wallet_address` rows described here, without this table itself knowing anything about signatures or chains.

## The entity

```ts
// src/components/wallet-address/entity/wallet-address.entity.ts
@Entity({ name: 'tbl_wallet_address' })
export class WalletAddressEntity extends BaseEntity {
    @Column({ nullable: false })
    public network: string;

    @Column({ unique: true, nullable: true })
    public walletAddress: string;

    @Column({ nullable: true })
    public userId: string;

    @Column({ nullable: true })
    public nonce: string;

    @ManyToOne(() => User, (user) => user.walletAddresses, { onDelete: 'CASCADE' })
    @JoinColumn({ name: 'userId' })
    public user: User;

    @Column({ nullable: true })
    freenameRegistrantUuid: string | null;
}
```

One user can have many wallet rows, `User.walletAddresses` is a `@OneToMany`, and in practice a single account can hold one row per chain family, an `evm_network` row and a `solana` row side by side (see `createSolanWithWallet` in the user service, which creates both at once for a Solana signup). `onDelete: 'CASCADE'` means deleting a `User` row deletes its wallet rows with it at the database level, not something the application code has to remember to clean up itself.

`walletAddress` is `unique` but `nullable`, the same pattern as `User.email`, and it's what makes the placeholder rows described in the previous note possible. `UserService.create` and `createWithGoogle` both insert a wallet row with `walletAddress: null` immediately after creating the user, so a brand-new email or Google account already has a row in this table before it has ever touched a wallet. Postgres's unique index doesn't treat two `NULL`s as a collision, so an unlimited number of these empty placeholder rows can coexist without ever tripping the constraint, it only engages once two rows try to claim the same real, non-null address.

`nonce` is the one-time value a wallet-signature login flow issues and checks off against, this table is where that value actually lives between being issued and being consumed, even though generating and verifying the signature itself happens in the auth cluster. `freenameRegistrantUuid` is a foreign identifier tying a wallet address to a registrant record in whatever third-party or on-chain registrar system "Freename" refers to, and `updateFreenameRegistrantUuid` and `findEvmWalletByRegistrantUuid` in the repository exist purely to let the blockchain/registrar integration cluster look a wallet up by that id, or attach it, without needing to know anything else about this table.

## What the repository actually supports

```ts
// src/components/wallet-address/wallet.repo.ts
async findByWallet(walletAddress: string, network: string): Promise<WalletAddressEntity> {
    return await this.WalletRepository.findOne({
        where: { walletAddress, network },
        select: ['walletAddress', 'network', 'nonce', 'userId']
    });
}
```

Nearly every lookup here is scoped by `network` as well as by wallet address or `userId`, because the same address string is theoretically not guaranteed unique across chains the way the column constraint enforces it globally, addresses on different networks are just distinguished by looking up the row with both fields. `findByWalletWithoutNetwork` is the deliberate exception, used specifically by `UserService.getByWallet` and `updateNonce`, where the caller genuinely doesn't know which network a given address belongs to yet and needs the row to tell them.

Every method that talks to the repository wraps its query in a `try/catch` that logs through `CustomLoggerService` before rethrowing, which is a defensive pattern worth noticing precisely because it's inconsistent, some methods on this same class (`findAllWalletAddress`, `findWithWalletAddrssAndUserId`) skip the try/catch entirely. Not a bug, just a sign of incremental hardening applied unevenly across one file over time.

## The service-layer methods that lean on this table

`UserService` (covered fully in the previous note) is where this table's data actually gets used. Worth knowing the shape of a few of these, since they're the pieces a frontend engineer would actually call into indirectly through the auth flow:

`getNonceByWalletAddress` and `getNonceById` are what a wallet-login flow reads from before asking a user to sign a message, the nonce has to already exist in this table for the signature challenge to be meaningful. `updateNonce`, `updateNonceWithUserId`, and `updateNonceForEmailLogin` are the three different ways a fresh nonce gets written back after a login attempt, split across three near-duplicate methods depending on whether the caller already has a `userId`, only a wallet address, or is coming from an email-based flow instead of a wallet one. `addWalletIntoUserAccount` is what lets an already-logged-in user link an additional wallet to their existing account rather than creating a new one, and `checkSingleSolanaAddress` specifically guards against a user's Solana wallet silently changing underneath them, if a Solana wallet is already on file and the caller supplies a different address, it throws rather than overwriting it.

None of this table's logic understands blockchain signatures, RPC calls, or on-chain state, it is deliberately just rows and lookups, that boundary is what makes it usable from both the auth side (who is this login attempt for) and the blockchain integration side (which address maps to this registrant) without either one having to import the other.
