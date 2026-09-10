# 06. What a User Actually Owns, and the One Route Getting an Extra Rate Limit

## The two views of "my domains"

`DomainDetailController`, at `domain/detail`, is what powers the account area where a signed in user sees the domains they hold, `GET /` for their own view and `GET /marketplace` for a public, wallet hash based view of the same list used when someone shares their portfolio. Both routes ultimately call the same service method, `getDomainListByUserId`, which reads from `DomainDetailBCRepo` against `tbl_domain_detail_bc`, the on chain mirror table described in the entities note, not the transactional purchase table. This is deliberate, a user's actual domain portfolio should reflect everything their wallet really holds right now, including names bought elsewhere, not just what was purchased through this company's own checkout.

## The `refresh_domain` route, and what it actually does

```ts
// src/components/domain/domain-detail/domain-detail.controller.ts
@ApiBearerAuth('defaultBearerAuth')
@Get('/refresh_domain')
@UseGuards(AccessTokenGuard)
public async refreshDomainDetails(@Req() req: Request): Promise<Response> {
    const userId = req.user['userId'];
    return new Response(DomainDetailSuccessMessage.DOMAIN_FETCH_SUCCESS, await this.domainDetailService.refreshDomainDetailData(userId));
}
```

And this is exactly the route `main.ts` singles out for a tighter, independent rate limit, right alongside forgot password:

```ts
// src/main.ts
app.use('/api/v1/auth/forgot-password', apiCallLimiter);
app.use('/api/v1/domain/detail/refresh_domain', apiCallLimiter);
```

To understand why this specific route earns that treatment, and not, say, the plain domain list endpoint sitting right next to it in the same controller, you have to read what `refreshDomainDetailData` actually does when it is called.

```ts
async refreshDomainDetailData(userId: string): Promise<any> {
    const response = await this.userRepo.findById(userId);
    const evmNetworkWallet = response.walletAddresses.find(wallet => wallet.network === 'evm_network');
    const solanaWalletAddresses = response.walletAddresses.find(wallet => wallet.network === 'solana');
    ...
    if (evmNetworkWallet.walletAddress) {
        const freenameAlchmey = await this.freenameAlchemyService.getAllNFTsOwnedbyAddress(blockchains, evmNetworkWallet.walletAddress, userId, evmNetworkWallet.freenameRegistrantUuid);
        const ARBdata = await this.AlchemysService.getAllNFTsOwnedbyAddress(DomainProvider.Arbitrum, evmNetworkWallet.walletAddress, userId);
        const BNBdata = await this.bnbAlchemyService.getAllNFTsOwnedbyAddress(DomainProvider.BinanceSmartChain, evmNetworkWallet.walletAddress, userId);
        const UDBasedata = await this.udBaseAlchemyService.getAllNFTsOwnedbyAddress(DomainProvider.UNSTOPPABLE_DOMAINS, evmNetworkWallet.walletAddress, userId);
        const ENSAlchmeydata = await this.ensAlchemyService.getAllNFTsOwnedbyAddress(DomainProvider.ENS, evmNetworkWallet.walletAddress, userId);
        const UDAlchmeyData = await this.udAlchemyService.getAllNFTsOwnedbyAddress(DomainProvider.UNSTOPPABLE_DOMAINS, evmNetworkWallet.walletAddress, userId);
        data.push(...BNBdata, ...ENSAlchmeydata, ...UDBasedata, ...UDAlchmeyData, ...ARBdata, ...freenameAlchmey);
    }
    if (solanaWalletAddresses) {
        const SolanaData = await this.MoralisService.nftsOwnedbySolanaWalletAddress(solanaWalletAddresses.walletAddress, userId);
        if (SolanaData.length > 0) { data.push(...SolanaData); }
    }
    await this.bulkUpdateDomainDetailBlockChain(data, userId);
    this.eventEmitter.emit('domain.sync.completed', new DomainSyncCompletedEvent(userId, data));
    ...
}
```

This is the single most expensive read triggered anywhere in this module. One call to this endpoint fires off, in sequence, six separate real time calls to Alchemy's NFT ownership API, one each for Freename, Arbitrum, BNB, the UD on Base variant, ENS, and UD's own primary chain, plus a seventh call to Moralis for Solana if the user has a Solana wallet on file, every single one of them an actual network round trip to a third party blockchain data provider, not a cached lookup or a local database query. Compare this against the plain `GET /` domain list route right above it in the same controller, which only ever reads from Postgres. That is the entire reason this specific route needed its own tighter limiter independent of the global sixty requests per minute `ThrottlerModule` limit set in `app.module.ts`, a user (or, more realistically, a script) hammering this endpoint repeatedly would multiply real cost and real latency against Alchemy and Moralis six or seven times over on every single call, and could plausibly exhaust this company's own rate limits with those providers, a cost and a risk no other route in this module carries at anything close to this scale.

## What happens once the data comes back

Once every provider has answered, `bulkUpdateDomainDetailBlockChain` upserts the fetched NFTs into `tbl_domain_detail_bc`, replacing that user's stored snapshot with what their wallets genuinely hold right now. Immediately after that succeeds, an internal event is emitted:

```ts
this.eventEmitter.emit('domain.sync.completed', new DomainSyncCompletedEvent(userId, data));
```

This is not a dead end, it is the entry point into the domain tenure system covered in the next note, one refresh here quietly kicks off a second, separate background process that tracks how long the user has actually held each of these domains. The refresh endpoint itself then simply re reads the first page of the user's now updated domain list and returns it.

## Other things this controller exposes

The rest of `DomainDetailController` is a smaller supporting cast around this same underlying data, `available-tlds` returns the distinct set of TLDs a given user actually owns something in, `user-analytic` powers a dashboard of a user's own portfolio broken down by TLD, blockchain, EVM versus non EVM, expired versus not, and listed on the marketplace versus not, `user-nfts/:walletAddress` is a thin passthrough to `UdAlchemyService.getAllNftsForAddress` for a raw NFT lookup by wallet, and `user/user-stats/:userId` is an admin facing summary, total domains, how many are expiring soon, how many are already expired, how many are listed for resale, and how many are not configured yet, computed with straightforward SQL aggregation and date math against `tbl_domain_detail_bc` rather than another live blockchain call, which is exactly why that one, unlike the refresh route, needs no special protection.
