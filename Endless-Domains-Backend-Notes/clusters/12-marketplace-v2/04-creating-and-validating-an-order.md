# 04. Creating and Validating an Order

## The shape of the problem

File `03` explained that a v2 listing is nothing more than a signed Seaport order sitting in `tbl_marketplacev2_orders`. That makes the create endpoint the single most security sensitive write in the whole marketplace. Anything this endpoint accepts becomes something the backend advertises to buyers, and because Seaport is a public contract, anything with a valid signature can be filled by anyone whether the backend likes it or not. The backend cannot stop a bad order from existing; it can only refuse to store and promote it. So the create path is designed as a gauntlet of twelve numbered checks, run in a fixed order, where the first failure throws a `400` with a machine readable reject code, and only an order that survives every one is written to the database.

The twelve checks were built in sprints and the code still shows the seams, which is actually helpful for learning: checks 1 to 7 ("local") need nothing but the request and the config, checks 8 to 10 ("chain") need RPC reads, and checks 11 and 12 ("persistence") need the database. Each layer has its own service method and, during development, its own diagnostic route.

```ts
// src/components/marketplacev2/order/order.service.ts (call graph, simplified)
create(dto, walletAddress, userId)
  -> validateChain(dto, walletAddress)
       -> validateLocal(dto, walletAddress)       // checks 1 to 7, fail fast
       -> Promise.all([checkCounter, checkOwnershipAndName, checkApproval])  // checks 8 to 10
  -> expireIfStale(tokenContract, tokenId)        // clears this token's own ghost row
  -> findOne(active and unexpired for this token) // check 11 ALREADY_LISTED
  -> transaction { insert order, upsertListed, history LISTED }  // check 12 DUPLICATE_ORDER on 23505
```

## The route and its two guards

```ts
// src/components/marketplacev2/order/order.controller.ts:224
@ApiBearerAuth('defaultBearerAuth')
@Post()
@UseGuards(AccessTokenGuard, RequireVerifiedWalletGuard)
async createOrder(@Body() dto: CreateOrderDto, @Req() req: Request): Promise<Response> {
    return new Response('Order created', await this.orderService.create(dto, req.walletAddress, req.user['userId']));
}
```

With the global prefix from `main.ts:58`, the full path is `POST /api/v1/marketplacev2/orders`. The controller is a textbook thin dispatcher in the house style: DTO in, one service call, wrapped in `new Response(message, result)`. Note the service is injected by the token `'OrderServiceInterface'` (`order.module.ts:31`), per the repo wide convention.

Guards run left to right. `AccessTokenGuard` is the standard Passport `jwt` guard; it verifies the Bearer access token and puts its payload on `req.user`, so `req.user['userId']` is the logged in user. `RequireVerifiedWalletGuard` (`src/@core/common/guards/require-verified-wallet.guard.ts`) is new in v2, and it is the reason the create path can trust a wallet address at all:

```ts
// src/@core/common/guards/require-verified-wallet.guard.ts
export const WALLET_VERIFICATION_WINDOW_MS = 60 * 60 * 1000;

canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const user = request.user as Express.User | undefined;
    const walletAddress = user?.walletAddress;
    const walletVerifiedAt = user?.walletVerifiedAt;

    const isVerified =
        !!walletAddress &&
        typeof walletVerifiedAt === 'number' &&
        Date.now() - walletVerifiedAt <= WALLET_VERIFICATION_WINDOW_MS;

    if (!isVerified) {
        throw new ForbiddenException({
            code: WalletVerificationErrorCode.WALLET_NOT_VERIFIED,
            message: 'Wallet verification is missing or has expired for this session.'
        });
    }

    request.walletAddress = walletAddress;
    return true;
}
```

The `walletAddress` and `walletVerifiedAt` claims are minted into the JWT by the wallet verification flow (`marketplacev2/wallet-verification/wallet-verification.service.ts:84`, covered elsewhere in this cluster), after the user has signed a nonce proving they control the wallet. This guard checks both claims exist and that the proof is at most an hour old, then copies the address onto `req.walletAddress`, which is what the controller passes down. The comment above the constant records a product decision worth knowing: the window was originally fifteen minutes, deliberately shorter than the token, and on 15 September 2026 it was widened to match the one hour access token lifetime at product's request. If verification is missing or stale, the frontend gets `403` with body `{ "code": "WALLET_NOT_VERIFIED", "message": "..." }` and should send the user back through wallet verification.

Why two separate identities? Because a user account and a wallet are different things in this app. A user can log in with email and password and have several wallets linked. The marketplace needs to know both who is acting (`userId`, for the listing status and history tables) and which wallet they have just proven control of (`walletAddress`, which must equal the order's offerer).

## The request body, field by field

```ts
// src/components/marketplacev2/order/dto/create-order.dto.ts
export class CreateOrderDto {
    @ValidateNested()
    @Type(() => OrderParametersDto)
    parameters: OrderParametersDto;

    @IsString()
    signature: string;

    @IsString()
    domainName: string;
}
```

The body has exactly three top level fields: the signed struct in `parameters`, the hex `signature`, and the `domainName` the seller claims this token is. `@Type(() => OrderParametersDto)` tells class transformer to instantiate the nested class so `@ValidateNested()` can run its validators; without `@Type`, nested validation silently does nothing, a classic NestJS footgun.

| Field | Validators | Notes |
|---|---|---|
| `parameters` | `@ValidateNested()`, `@Type(() => OrderParametersDto)` | The `OrderComponents` struct with every number as a decimal string. |
| `signature` | `@IsString()` | No format check here; a malformed value is caught by check 2. |
| `domainName` | `@IsString()` | No format or emptiness check; verified against the database by check 9. |

```ts
// src/components/marketplacev2/order/dto/order-parameters.dto.ts
export class OrderParametersDto {
    @Matches(ETH_ADDRESS_PATTERN) offerer: string;
    @Matches(ETH_ADDRESS_PATTERN) zone: string;
    @IsArray() @ValidateNested({ each: true }) @Type(() => OfferItemDto) offer: OfferItemDto[];
    @IsArray() @ValidateNested({ each: true }) @Type(() => ConsiderationItemDto) consideration: ConsiderationItemDto[];
    @IsIn(Object.values(OrderType)) orderType: number;
    @IsNumberString() startTime: string;
    @IsNumberString() endTime: string;
    @Matches(BYTES32_PATTERN) zoneHash: string;
    @IsNumberString() salt: string;
    @Matches(BYTES32_PATTERN) conduitKey: string;
    @IsNumberString() counter: string;
}
```

| Field | Validators | Meaning |
|---|---|---|
| `offerer` | `@Matches(ETH_ADDRESS_PATTERN)` | Seller wallet. |
| `zone` | `@Matches(ETH_ADDRESS_PATTERN)` | Must be the zero address (check 6). |
| `offer` | `@IsArray()`, `@ValidateNested({ each: true })`, `@Type(() => OfferItemDto)` | Must have exactly one item (check 3). |
| `consideration` | `@IsArray()`, `@ValidateNested({ each: true })`, `@Type(() => ConsiderationItemDto)` | Must have exactly two items (check 4). |
| `orderType` | `@IsIn(Object.values(OrderType))` | A JSON number; must be `0` (check 6). |
| `startTime` | `@IsNumberString()` | Unix seconds. |
| `endTime` | `@IsNumberString()` | Unix seconds. |
| `zoneHash` | `@Matches(BYTES32_PATTERN)` | Must be all zeros (check 6). |
| `salt` | `@IsNumberString()` | Decimal string of the 256 bit salt. |
| `conduitKey` | `@Matches(BYTES32_PATTERN)` | Must be all zeros (check 6). |
| `counter` | `@IsNumberString()` | Must equal the on chain counter (check 8). |

```ts
// src/components/marketplacev2/order/dto/offer-item.dto.ts and consideration-item.dto.ts
export class OfferItemDto {
    @IsIn(Object.values(ItemType)) itemType: number;
    @Matches(ETH_ADDRESS_PATTERN) token: string;
    @IsNumberString() identifierOrCriteria: string;
    @IsNumberString() startAmount: string;
    @IsNumberString() endAmount: string;
}
export class ConsiderationItemDto {
    // the same five fields and validators, plus:
    @Matches(ETH_ADDRESS_PATTERN) recipient: string;
}
```

```ts
// src/components/marketplacev2/order/dto/eth-format.constants.ts
/** Unchecksummed hex is accepted here; checks compare via ethers.getAddress() downstream. */
export const ETH_ADDRESS_PATTERN = /^0x[a-fA-F0-9]{40}$/;
export const BYTES32_PATTERN = /^0x[a-fA-F0-9]{64}$/;
```

Two design choices deserve explanation. Every big number travels as a decimal string validated by `@IsNumberString()`, never as a JSON number, because JSON numbers become JavaScript doubles and would corrupt a `uint256` token id or a salt. And addresses are accepted in any letter case, because wallets and libraries disagree about checksumming; the service normalises everything through `ethers.getAddress()` before comparing, so the comparison is always checksummed against checksummed.

The global pipe in `main.ts:62` is `new ValidationPipe({ whitelist: true, transform: true })`. `whitelist` strips any property not declared on the DTO (so a client cannot smuggle `status: 'filled'` or `orderHash` in), and `transform` builds real class instances. If any validator fails, Nest returns its standard `400` before the service runs, with a body like `{ "statusCode": 400, "message": ["parameters.offer.0.startAmount must be a number string"], "error": "Bad Request" }`. That shape differs from the reject code shape below (no `code` field, `message` is an array), so the frontend has to handle both.

## Turning strings into a real order

Before any check, the DTO is converted into the shared package's `OrderComponents` type, with addresses checksummed and numbers turned into `bigint`:

```ts
// src/components/marketplacev2/order/order.service.ts:93
function toOrderComponents(dto: OrderParametersDto): OrderComponents {
    try {
        return {
            offerer: getAddress(dto.offerer),
            zone: getAddress(dto.zone),
            offer: dto.offer.map((item) => ({
                itemType: item.itemType,
                token: getAddress(item.token),
                identifierOrCriteria: BigInt(item.identifierOrCriteria),
                startAmount: BigInt(item.startAmount),
                endAmount: BigInt(item.endAmount)
            })),
            // consideration mapped the same way, plus recipient: getAddress(item.recipient)
            orderType: dto.orderType,
            startTime: BigInt(dto.startTime),
            endTime: BigInt(dto.endTime),
            zoneHash: dto.zoneHash,
            salt: BigInt(dto.salt),
            conduitKey: dto.conduitKey,
            counter: BigInt(dto.counter)
        };
    } catch (err) {
        reject(OrderRejectCode.BAD_OFFER_ITEM, `Order parameters could not be parsed: ${err instanceof Error ? err.message : String(err)}`);
    }
}
```

The comment admits the `catch` is "unreachable in practice" since the DTO validators already caught malformed hex, but keeps it "as a hard boundary rather than letting a bad numeric string reach ethers as a 500." Labelling a generic parse failure `BAD_OFFER_ITEM` is slightly misleading (a bad `counter` string would report as an offer problem), but since it cannot really fire it is harmless.

Every rejection goes through one tiny helper, and it is worth seeing exactly what it produces:

```ts
// src/components/marketplacev2/order/order.service.ts:83
function reject(code: OrderRejectCode, message: string): never {
    throw new BadRequestException({ code, message });
}
```

When you pass an object to a Nest HTTP exception, Nest sends that object as the entire response body. There is no global exception filter in this app, so the frontend receives HTTP `400` with exactly `{ "code": "FEE_MISMATCH", "message": "Consideration split (seller ..., fee ...) does not match ..." }`, without the usual `statusCode` and `error` keys. The `never` return type is a neat TypeScript trick: it tells the compiler control never continues past a `reject(...)` call, which is what lets code like `let recovered: string; try { recovered = ... } catch { reject(...) }` compile without "used before assigned" errors.

## The local checks, one through seven

`validateLocal` (`order.service.ts:181`) runs these in order and stops at the first failure. Because it never touches the network or the database, it is cheap, which is why it always runs first: there is no point paying for three RPC calls on an order with a forged signature.

### Check 1, OFFERER_NOT_CALLER

```ts
// src/components/marketplacev2/order/order.service.ts:185
if (components.offerer !== getAddress(walletAddress)) {
    reject(OrderRejectCode.OFFERER_NOT_CALLER, `Order offerer ${components.offerer} does not match the verified wallet ${walletAddress}.`);
}
```

You may only list as yourself. Without this check, anyone who obtained someone else's signed order (say, from the public `GET /:orderHash` route after it was cancelled and relisted elsewhere) could submit it under their own account. Both sides are checksummed, so letter case cannot cause a false mismatch.

### Check 2, BAD_SIGNATURE

```ts
// src/components/marketplacev2/order/order.service.ts:190
const digest = computeDigest(components, BigInt(MARKETPLACEV2_CHAIN_ID), this.chainConfig.seaportAddress);
let recovered: string;
try {
    recovered = recoverAddress(digest, dto.signature);
} catch (err) {
    reject(OrderRejectCode.BAD_SIGNATURE, `Signature could not be recovered: ${err instanceof Error ? err.message : String(err)}`);
}
if (recovered !== components.offerer) {
    reject(OrderRejectCode.BAD_SIGNATURE, `Signature recovers to ${recovered}, expected offerer ${components.offerer}.`);
}
```

This is how signature verification works, in three steps. First, the backend rebuilds the EIP712 digest itself from the submitted fields, with the chain id hard coded to `137` and the verifying contract taken from config, never from the request. Second, `recoverAddress` (ethers v6) takes the digest and the 65 or 64 byte signature and computes which public key, and therefore which address, produced it; a string that is not a parseable signature at all throws, and is reported as "could not be recovered." Third, the recovered address must equal the offerer. If the frontend signed with a different Seaport address, a different chain id, a different field order, or if anyone altered a single field after signing, the digest differs and the recovered address is a meaningless stranger, so the order is rejected.

Note what this means for wallet types: `recoverAddress` only works for externally owned accounts (MetaMask style private keys). A smart contract wallet such as a Safe signs via EIP1271, which needs an on chain `isValidSignature` call, and Seaport supports that, but this check does not, so a contract wallet seller cannot list. That is a reasonable scope decision for now, just one to know about.

### Check 3, BAD_OFFER_ITEM

```ts
// src/components/marketplacev2/order/order.service.ts:202
if (components.offer.length !== 1) {
    reject(OrderRejectCode.BAD_OFFER_ITEM, `Expected exactly 1 offer item, got ${components.offer.length}.`);
}
const [offerItem] = components.offer;
if (offerItem.itemType !== ItemType.ERC721) { reject(OrderRejectCode.BAD_OFFER_ITEM, /* ... */); }
if (offerItem.startAmount !== 1n || offerItem.endAmount !== 1n) {
    reject(OrderRejectCode.BAD_OFFER_ITEM, 'ERC-721 offer amount must be exactly 1.');
}
if (offerItem.token !== getAddress(this.chainConfig.domainNftAddress)) {
    reject(OrderRejectCode.BAD_OFFER_ITEM, `Offer token ${offerItem.token} is not the configured domain NFT contract.`);
}
```

Four rejections share this code: not exactly one offer item (no bundles, no empty offers), not an ERC721, an amount other than one, or a token contract other than the configured domain NFT. The last is what stops someone using the Endless Domains marketplace to advertise an unrelated NFT collection.

### Check 4, BAD_CONSIDERATION

```ts
// src/components/marketplacev2/order/order.service.ts:217
if (components.consideration.length !== 2) { reject(OrderRejectCode.BAD_CONSIDERATION, /* ... */); }
const [sellerLeg, feeLeg] = components.consideration;
for (const [index, item] of components.consideration.entries()) {
    if (item.itemType !== ItemType.ERC20) { reject(/* ... expected ERC20 */); }
    if (item.token !== getAddress(this.chainConfig.usdtAddress)) { reject(/* ... not the configured USDT contract */); }
    if (item.identifierOrCriteria !== 0n) { reject(/* ... must be 0 for an ERC20 */); }
    if (item.startAmount !== item.endAmount) { reject(/* ... has a price curve; a flat amount is required */); }
}
if (sellerLeg.recipient !== components.offerer) {
    reject(OrderRejectCode.BAD_CONSIDERATION, `consideration[0].recipient ${sellerLeg.recipient} must be the offerer.`);
}
```

Six rejections: wrong number of legs, a leg that is not ERC20, a leg not paid in the configured USDT, a non zero identifier on an ERC20 leg, a price curve (start not equal to end), and leg zero not paid to the offerer. The ordering convention matters: leg zero is always the seller and leg one is always the fee. The fee leg's recipient is checked in check 5, not here.

### Check 5, FEE_MISMATCH

```ts
// src/components/marketplacev2/order/order.service.ts:240
const total = sellerLeg.startAmount + feeLeg.startAmount;
let split: ReturnType<typeof computeSplit>;
try {
    split = computeSplit(total, BigInt(this.chainConfig.feeBps));
} catch (err) {
    if (err instanceof OrderError) {
        reject(OrderRejectCode.FEE_MISMATCH, err.message);
    }
    throw err;
}
if (feeLeg.startAmount !== split.fee || sellerLeg.startAmount !== split.sellerAmount) {
    reject(OrderRejectCode.FEE_MISMATCH, /* ... does not match the configured fee policy ... */);
}
if (feeLeg.recipient !== getAddress(this.chainConfig.feeRecipient)) {
    reject(OrderRejectCode.FEE_MISMATCH, `Fee recipient ${feeLeg.recipient} is not the configured fee recipient.`);
}
if (split.fee === 0n) {
    reject(OrderRejectCode.FEE_MISMATCH, 'The fee split for this price rounds down to zero; the listing price is too low for the configured fee policy.');
}
if (split.sellerAmount === 0n) {
    reject(OrderRejectCode.FEE_MISMATCH, 'The seller split for this price rounds down to zero.');
}
```

The trick here is that the backend never asks the client "what is the price." It derives the total by adding the two signed legs, recomputes what the split should be for that total at the configured `FEE_BPS`, and demands the signed legs match exactly. A seller who signs 90 to themselves and 10 to the fee wallet on a 100 total is rejected, because 2.5 percent of 100 is 2.5, not 10. Five rejections share the code: `computeSplit` throwing an `OrderError` (bad input to the package), the legs not matching the policy split, the fee leg paid to anyone but `FEE_RECIPIENT`, the fee rounding to zero (prices under 40 minor units at 250 basis points, since 39 times 250 divided by 10000 truncates to 0), and the seller share rounding to zero (only reachable if `FEE_BPS` were `10000`). The zero checks exist because Seaport reverts on a zero amount consideration item, so such an order would sit in browse forever looking buyable while being impossible to buy. A non `OrderError` from `computeSplit` is rethrown and becomes a `500`.

### Check 6, BAD_ORDER_TYPE

```ts
// src/components/marketplacev2/order/order.service.ts:268
if (components.orderType !== OrderType.FULL_OPEN) { reject(/* ... is not FULL_OPEN */); }
if (components.zone !== ZERO_ADDRESS) { reject(OrderRejectCode.BAD_ORDER_TYPE, 'FULL_OPEN orders must not set a zone.'); }
if (components.zoneHash !== ZERO_BYTES32) { reject(OrderRejectCode.BAD_ORDER_TYPE, 'zoneHash must be zero for a FULL_OPEN order with no zone.'); }
if (components.conduitKey !== NO_CONDUIT_KEY) {
    reject(OrderRejectCode.BAD_ORDER_TYPE, 'conduitKey must be the no-conduit key; the seller approves Seaport directly.');
}
```

Four rejections, all about locking the order to the simplest Seaport shape. `zoneHash` and `conduitKey` are compared as raw strings, which is why uppercase hex zeros would not be an issue (zero has no letters) but any nonzero value is caught.

### Check 7, BAD_TIME_WINDOW

```ts
// src/components/marketplacev2/order/order.service.ts:79
const MAX_START_TIME_AHEAD_SECONDS = 300n; // 5 minutes - clock skew tolerance
const MIN_REMAINING_SECONDS = 60n * 60n; // 1 hour
const MAX_REMAINING_SECONDS = 90n * 24n * 60n * 60n; // 90 days

// src/components/marketplacev2/order/order.service.ts:290
const nowSeconds = BigInt(Math.floor(Date.now() / 1000));
if (components.startTime >= components.endTime) { reject(/* 'startTime must be before endTime.' */); }
if (components.startTime > nowSeconds + MAX_START_TIME_AHEAD_SECONDS) { reject(/* startTime is too far in the future */); }
if (nowSeconds > components.endTime) { reject(/* Order has expired */); }
const remainingSeconds = components.endTime - nowSeconds;
if (remainingSeconds < MIN_REMAINING_SECONDS) { reject(/* Order expires too soon */); }
if (remainingSeconds > MAX_REMAINING_SECONDS) { reject(/* Order window extends too far into the future */); }
```

Five rejections: an inverted or empty window, a start more than five minutes in the future, an already ended window, less than one hour of life left, and more than ninety days of life left. The long comment above this block explains each bound. The five minute start tolerance exists because the builder backdates `startTime` by 300 seconds, so a legitimate order is never meaningfully in the future, and an earlier version that rejected any future start at all was stricter than the brief (the spec "accepts a startTime a little in the future" guards that fix). The one hour floor stops accepting orders that will die before anyone sees them. The ninety day ceiling is, in the comment's words, the ghost listing bound the brief is explicitly guarding against: the longer an order lives, the more likely the seller has forgotten it and the world has changed under it. There is no lower bound on `startTime`; an order signed with a start in 1970 is fine, since Seaport only cares that it has started.

### What validateLocal returns

```ts
// src/components/marketplacev2/order/order.service.ts:308
return {
    orderHash: computeOrderHash(components),
    offerer: components.offerer,
    tokenContract: offerItem.token,
    tokenId: offerItem.identifierOrCriteria.toString(),
    priceUsdt: total.toString(),
    feeUsdt: feeLeg.startAmount.toString(),
    sellerUsdt: sellerLeg.startAmount.toString(),
    startTime: components.startTime.toString(),
    endTime: components.endTime.toString()
};
```

This `OrderLocalValidationResult` (defined in `interface/order-service.interface.ts:19`) is also what `create()` finally returns to the client, so a successful `POST` responds with `{ "message": "Order created", "result": { orderHash, offerer, tokenContract, tokenId, priceUsdt, feeUsdt, sellerUsdt, startTime, endTime } }`, every amount a decimal string. The frontend should compare the returned `orderHash` with the one it computed locally; they must match.

## The chain checks, eight through ten

`validateChain` (`order.service.ts:416`) first calls `validateLocal`, and only if that succeeds does it touch the network. The spec titled `runs local checks 1-7 first and never touches the chain if they fail` proves no RPC method is called when local validation fails.

### The provider and the minimal ABIs

```ts
// src/components/marketplacev2/order/order.service.ts:159
this.provider = new JsonRpcProvider(this.chainConfig.polygonRpcUrl, MARKETPLACEV2_CHAIN_ID, { staticNetwork: true });
this.seaportContract = new Contract(this.chainConfig.seaportAddress, SEAPORT_MIN_ABI, this.provider);
this.domainNftContract = new Contract(this.chainConfig.domainNftAddress, ERC721_MIN_ABI, this.provider);
```

```ts
// src/components/marketplacev2/order/abi/chain-check.abi.ts
export const SEAPORT_MIN_ABI = ['function getCounter(address offerer) view returns (uint256 counter)'] as const;

export const ERC721_MIN_ABI = [
    'function ownerOf(uint256 tokenId) view returns (address)',
    'function isApprovedForAll(address owner, address operator) view returns (bool)'
] as const;
```

These are human readable ABI fragments, an ethers feature that lets you describe just the functions you call instead of importing a giant JSON ABI. The header comment notes the signatures were copied from the internal `seaport-polygon-spike` repo, which verified them against the live Seaport 1.6 deployment by checking selectors against deployed bytecode and calling the view functions directly, rather than deriving them again from documentation. All three are `view` functions, so they are free `eth_call` reads, never transactions.

The provider is built once in the constructor, not per request, so connections are reused. `staticNetwork: true` tells ethers v6 to trust the given chain id instead of probing `eth_chainId`; the comment explains that without it "a down/unreachable RPC makes the provider retry network detection every second forever."

### Check 8, STALE_COUNTER

```ts
// src/components/marketplacev2/order/order.service.ts:325
private async checkCounter(components: OrderComponents): Promise<ChainCheckResult> {
    let onChainCounter: bigint;
    try {
        onChainCounter = await retryWithBackoff(() => this.seaportContract.getCounter(components.offerer), { isRetryable: isRetryableChainError });
    } catch (err) {
        return { ok: false, kind: 'unavailable', message: `Could not read the offerer's counter from Seaport: ...` };
    }
    if (onChainCounter !== components.counter) {
        return { ok: false, kind: 'reject', code: OrderRejectCode.STALE_COUNTER, message: `On-chain counter ${onChainCounter} does not match the signed order's counter ${components.counter}.` };
    }
    return { ok: true };
}
```

If the seller has ever called Seaport's `incrementCounter()` since signing, this order is already dead on chain and storing it would create a ghost listing. Comparing `bigint` to `bigint` with `!==` is safe; ethers v6 returns `uint256` values as native `bigint`.

### Check 9, NOT_OWNER and NAME_MISMATCH

```ts
// src/components/marketplacev2/order/order.service.ts:353
private async checkOwnershipAndName(components: OrderComponents, domainName: string): Promise<ChainCheckResult> {
    const tokenId = components.offer[0].identifierOrCriteria;
    let owner: string;
    try {
        owner = await retryWithBackoff(() => this.domainNftContract.ownerOf(tokenId), { isRetryable: isRetryableChainError });
    } catch (err) {
        return { ok: false, kind: 'unavailable', message: `Could not read ownerOf(${tokenId}) from the domain NFT contract: ...` };
    }
    if (getAddress(owner) !== components.offerer) {
        return { ok: false, kind: 'reject', code: OrderRejectCode.NOT_OWNER, message: `tokenId ${tokenId} is owned by ${owner}, not the offerer ${components.offerer}.` };
    }

    const record = await this.domainDetailBCRepo.findByTokenId(tokenId.toString(), this.chainConfig.domainNftAddress);
    if (!record || record.domainName !== domainName) {
        return { ok: false, kind: 'reject', code: OrderRejectCode.NAME_MISMATCH, message: `tokenId ${tokenId} does not correspond to domain "${domainName}".` };
    }
    return { ok: true };
}
```

The ownership half is a real chain read: `ownerOf(tokenId)` must return the offerer. The name half is not, and the comment is admirably upfront about it: the domain NFT contract (`EndlessCollection.sol`) has no on chain function mapping a token id to a name, and its `tokenURI()` returns one URI shared by the whole collection, so the only ground truth is the backend's own ownership cache, `tbl_domain_detail_bc`, via `DomainDetailBCRepo.findByTokenId`:

```ts
// src/components/domain/domain-detail/domain-detail-bc.repo.ts:501
async findByTokenId(tokenId: string, registryAddress: string): Promise<DomainDetailBCEntity | null> {
    return this.repo
        .createQueryBuilder('d')
        .where('d.token_id = :tokenId', { tokenId })
        .andWhere('LOWER(d."registryAddress") = LOWER(:registryAddress)', { registryAddress })
        .andWhere('d."isDeleted" = false')
        .getOne();
}
```

Why does the name matter if the token id is what is actually sold? Because `domainName` is what buyers see. Without this check, a seller could list a worthless token under the label `google.crypto`, and buyers would pay for a name they never receive. A missing record is also a `NAME_MISMATCH` (the spec "rejects NAME_MISMATCH when no domain detail record exists for the tokenId" covers it), which in practice means a seller must have synced their domains into `tbl_domain_detail_bc` (for example via `GET /domain/detail/refresh_domain`) before they can list. The comparison is exact and case sensitive.

### Check 10, NOT_APPROVED

```ts
// src/components/marketplacev2/order/order.service.ts:385
approved = await retryWithBackoff(() => this.domainNftContract.isApprovedForAll(components.offerer, this.chainConfig.seaportAddress), { isRetryable: isRetryableChainError });
// ...
if (!approved) {
    return { ok: false, kind: 'reject', code: OrderRejectCode.NOT_APPROVED, message: `Offerer ${components.offerer} has not approved Seaport (${this.chainConfig.seaportAddress}) to transfer the domain NFT.` };
}
```

At fill time Seaport calls `transferFrom(seller, buyer, tokenId)` on the domain NFT. That only works if the seller has previously called `setApprovalForAll(seaport, true)`. Because the conduit key is forced to zero, the operator to check is Seaport itself. This is the one step in listing that costs the seller gas, once per wallet, and the frontend's listing flow sends that approval transaction before asking for the signature.

On balances, since the brief for these notes asked: the backend checks no ERC20 balances or allowances on create. That is correct rather than an omission. The seller is not paying anything; the buyer is, and the buyer is not known until fill time. The frontend's buy flow reads `balanceOf` and `allowance` on USDT (its `ERC20_MIN_ABI`) before filling.

### Running the three in parallel and choosing a winner

```ts
// src/components/marketplacev2/order/order.service.ts:420
const results: ChainCheckResult[] = await Promise.all([this.checkCounter(components), this.checkOwnershipAndName(components, dto.domainName), this.checkApproval(components)]);

let firstUnavailable: Extract<ChainCheckResult, { kind: 'unavailable' }> | undefined;
for (const result of results) {
    if (result.ok === false) {
        if (result.kind === 'reject') {
            reject(result.code, result.message);
        }
        firstUnavailable ??= result;
    }
}
if (firstUnavailable) {
    this.customLoggerService.error(`validateChain: chain read unavailable - ${firstUnavailable.message}`);
    throw new ServiceUnavailableException({ code: 'CHAIN_UNAVAILABLE', message: CHAIN_UNAVAILABLE_CLIENT_MESSAGE });
}
```

This is a nice pattern to steal. Each check returns a discriminated union (`{ ok: true }`, or `{ ok: false, kind: 'reject', code, message }`, or `{ ok: false, kind: 'unavailable', message }`) instead of throwing. That keeps two very different failures apart: "your order is wrong" (a `400` the user can fix) and "we could not reach the chain" (a `503` that is our problem). The three reads run concurrently with `Promise.all`, so the latency is the slowest one, not the sum. The results are then inspected in a fixed order (8, 9, 10), so the same inputs always produce the same error. Any `reject` beats any `unavailable`, on the reasoning that "if we already know the order is invalid, that's more useful to report."

The comment about `result.ok === false` rather than `!result.ok` is a real TypeScript quirk in this project's compiler settings: narrowing a union on a boolean literal discriminant only worked with strict equality, and the comment pleads with future readers not to "simplify" it.

The `503` message is deliberately generic: `'Chain temporarily unavailable. Please try again shortly.'`. The comment explains why: ethers v6 embeds a `requestUrl` in HTTP level provider errors, and "that URL is the paid RPC endpoint, key and all." The detailed message goes to the server logs (CloudWatch) only, never to the client.

### The retry predicate

All three reads go through `retryWithBackoff` with a chain specific predicate:

```ts
// src/@core/utils/retry/retry-with-backoff.util.ts
const DEFAULT_IS_RETRYABLE = (error: any): boolean => error?.code === 429 || error?.status === 'RESOURCE_EXHAUSTED' || error?.response?.status === 429;

export async function retryWithBackoff<T>(fn: () => Promise<T>, options: RetryOptions = {}): Promise<T> {
    const { maxRetries = 3, baseDelayMs = 500, isRetryable = DEFAULT_IS_RETRYABLE } = options;
    let attempt = 0;
    for (;;) {
        try {
            return await fn();
        } catch (error) {
            if (!isRetryable(error) || attempt >= maxRetries) {
                throw error;
            }
            const jitter = Math.random() * baseDelayMs;
            await new Promise((resolve) => setTimeout(resolve, baseDelayMs * 2 ** attempt + jitter));
            attempt++;
        }
    }
}
```

```ts
// src/@core/utils/retry/is-retryable-chain-error.util.ts
export function isRetryableChainError(error: unknown): boolean {
    const err = error as { code?: unknown; status?: unknown; response?: { status?: unknown } } | null | undefined;
    if (!err) return false;
    const code = typeof err.code === 'string' ? err.code : undefined;
    if (code && ['ECONNRESET', 'ETIMEDOUT', 'ECONNREFUSED', 'ENOTFOUND', 'TIMEOUT', 'NETWORK_ERROR', 'SERVER_ERROR'].includes(code)) {
        return true;
    }
    const status = typeof err.status === 'number' ? err.status : typeof err.response?.status === 'number' ? err.response.status : undefined;
    return typeof status === 'number' && status >= 500 && status < 600;
}
```

Up to four attempts in total (the first plus three retries), waiting about 500, 1000 and 2000 milliseconds plus up to 500 milliseconds of random jitter each, so roughly four to five and a half seconds of waiting in the worst case, on top of the RPC calls themselves. Jitter matters when many requests fail together: without it they would all retry at the same instant and hammer the node again.

The predicate retries Node socket errors, ethers v6's own `TIMEOUT`, `NETWORK_ERROR` and `SERVER_ERROR` codes, and any `5xx` status. Its comment explains why the generic default (which only retries `429` quota errors, written originally for a Google API) is "the wrong shape for an RPC node that's merely slow or briefly down." It was extracted into `src/@core/utils/retry/` in commit `546e26e9` so the poller's tick loop shares the exact same predicate. Crucially, it does not retry `CALL_EXCEPTION`, which is ethers' code for "the contract call reverted": a revert is a deterministic answer, and asking again will not change it. That is correct in principle, but see the bug list below for one place it bites.

## Checks eleven and twelve, and persistence

```ts
// src/components/marketplacev2/order/order.service.ts:642
async create(dto: CreateOrderDto, walletAddress: string, userId: string): Promise<OrderLocalValidationResult> {
    const result = await this.validateChain(dto, walletAddress);

    await this.expireIfStale(result.tokenContract, result.tokenId);

    const nowSeconds = Math.floor(Date.now() / 1000).toString();
    const alreadyListed = await this.orderRepository.findOne({
        where: { tokenContract: result.tokenContract, tokenId: result.tokenId, status: OrderStatus.ACTIVE, endTime: MoreThan(nowSeconds) }
    });
    if (alreadyListed) {
        reject(OrderRejectCode.ALREADY_LISTED, `tokenId ${result.tokenId} on ${result.tokenContract} already has an active listing (orderHash ${alreadyListed.orderHash}).`);
    }
    // ...
}
```

Before checking for an existing listing, `create()` calls `expireIfStale` for this exact token (explained fully in file `05`). Here is the bug that motivated it: the partial unique index only lets one `active` row exist per token, and nothing used to move a row out of `active` when its `endTime` passed. So a seller whose listing quietly expired could never relist, because the dead row still held the index slot, and they got a confusing `ALREADY_LISTED` until they manually cancelled a listing that was already dead. Expiring the token's own stale row first, in its own transaction, clears the slot.

Check 11 then looks for any row for this token that is `active` and whose `endTime` is still in the future. If one exists, it is a genuine live competing listing, almost always the seller's own previous listing of the same domain at a different price, and the right answer is "cancel the old one first." The message includes the existing order hash, which a frontend can use to offer a "cancel and replace" button.

```ts
// src/components/marketplacev2/order/order.service.ts:655
const [, tld] = await splitDomainNameAndTLD(dto.domainName);
const components = toOrderComponents(dto.parameters);

const entity = this.orderRepository.create({
    orderHash: result.orderHash,
    status: OrderStatus.ACTIVE,
    maker: result.offerer,
    tokenContract: result.tokenContract,
    tokenId: result.tokenId,
    chainId: MARKETPLACEV2_CHAIN_ID,
    priceUsdt: result.priceUsdt,
    feeUsdt: result.feeUsdt,
    sellerUsdt: result.sellerUsdt,
    startTime: result.startTime,
    endTime: result.endTime,
    counter: components.counter.toString(),
    salt: components.salt.toString(),
    conduitKey: components.conduitKey,
    signature: dto.signature,
    rawOrder: dto.parameters as unknown as Record<string, unknown>,
    domainName: dto.domainName,
    tld: tld ?? '',
    createdAt: new Date(),
    filledAt: null,
    fillTxHash: null,
    userId
});
```

What gets persisted: every column in file `03`'s table, with `status` `active`, the hash computed server side, `maker` and `tokenContract` checksummed, all amounts as decimal strings, the signature exactly as submitted, and `rawOrder` set to the validated `parameters` object. That last one is the struct the buyer will later replay to Seaport. It is "as received" with two caveats: the global `whitelist` already stripped any undeclared fields, and addresses inside it keep whatever letter case the client sent (harmless, because ABI encoding of an address ignores case). `filledBy` and `invalidReason` are not set and fall to `null` through the column defaults. `userId` comes straight from the JWT, which is the whole point of the column.

```ts
// src/components/marketplacev2/order/order.service.ts:687
try {
    await this.orderRepository.manager.transaction(async (manager) => {
        await manager.insert(OrderEntity, entity);
        await this.listingStatusService.upsertListed(manager, userId, dto.domainName, result.tokenId);
        await this.listingHistoryService.record(manager, {
            userId, domainName: dto.domainName, tokenId: result.tokenId, tokenContract: result.tokenContract,
            orderHash: result.orderHash, eventType: ListingEventType.LISTED, txHash: null, occurredAt: new Date()
        });
    });
} catch (err) {
    const dbErr = err as { code?: string; constraint?: string };
    if (dbErr?.code === PostgresErrorCode.UNIQUE_VIOLATION) {
        if (dbErr.constraint === 'idx_marketplacev2_orders_active_listing_unique') {
            reject(OrderRejectCode.ALREADY_LISTED, `tokenId ${result.tokenId} on ${result.tokenContract} already has an active listing (lost a race with a concurrent submission).`);
        }
        reject(OrderRejectCode.DUPLICATE_ORDER, `Order ${result.orderHash} has already been submitted.`);
    }
    throw err;
}
```

Three writes, one transaction. The order row, the `tbl_marketplacev2_listing_status` flip to `LISTED` for this user, domain and token (an upsert on its `(userId, domainName, tokenId)` unique key), and an append only `LISTED` row in `tbl_marketplacev2_listing_history` (inserted with `orIgnore()`, so a replay never duplicates it). The comment states the rule: "the status/history tables must never disagree with the order table about whether this listing actually happened." Either all three land or none do.

The insert comment tells a good debugging story. The original code used `save()`, and TypeORM's `save()` is an upsert: given an entity whose primary key already exists, it issues an `UPDATE`. So resubmitting a cancelled order's exact same signature silently flipped it back to `active`, and the `DUPLICATE_ORDER` branch "was unreachable dead code." `insert()` always issues a real `INSERT`, so a repeat order hash actually collides with the primary key. This is a lesson worth remembering: `save()` is convenient but it hides whether you created or updated.

The error mapping distinguishes the two unique constraints that can fire. Postgres error `23505` (`PostgresErrorCode.UNIQUE_VIOLATION`) with constraint `idx_marketplacev2_orders_active_listing_unique` means two different orders for the same token raced each other past check 11, and the database refused the second; that is reported as `ALREADY_LISTED`, "not a misleading DUPLICATE_ORDER." Any other `23505` is taken to be the primary key, `DUPLICATE_ORDER`, meaning this exact order hash is already stored (perhaps the user double clicked, or the frontend retried after a timeout). Any other database error is rethrown and becomes a `500`. TypeORM's `QueryFailedError` copies the driver's `code` and `constraint` onto itself, which is why reading them off the caught error works.

## Every error the frontend can receive from POST /orders

| HTTP | Body `code` | Check | Exact trigger |
|---|---|---|---|
| 401 | none (Passport) | guard | Missing, malformed or expired access token. |
| 403 | `WALLET_NOT_VERIFIED` | guard | No `walletAddress` or `walletVerifiedAt` claim in the JWT, or the verification is more than one hour old. |
| 400 | none (`message` array) | DTO | Any class validator failure: a bad address or bytes32 pattern, a non numeric string, a non array `offer` or `consideration`, an `itemType` or `orderType` outside the package's enum values, a missing `signature` or `domainName`. |
| 400 | `BAD_OFFER_ITEM` | parse | Parameters could not be converted to `bigint` or checksummed (practically unreachable). |
| 400 | `OFFERER_NOT_CALLER` | 1 | `parameters.offerer` is not the verified wallet. |
| 400 | `BAD_SIGNATURE` | 2 | Signature cannot be parsed, or recovers to an address other than the offerer. |
| 400 | `BAD_OFFER_ITEM` | 3 | Not exactly one offer item; not ERC721; amount not exactly one; token not `DOMAIN_NFT_ADDRESS_POLYGON_UD`. |
| 400 | `BAD_CONSIDERATION` | 4 | Not exactly two legs; a leg not ERC20; not the configured USDT; nonzero identifier; start amount not equal to end amount; leg zero not paid to the offerer. |
| 400 | `FEE_MISMATCH` | 5 | `computeSplit` raised an `OrderError`; legs do not equal the policy split; fee leg not paid to `FEE_RECIPIENT`; fee rounds to zero (price too low); seller share rounds to zero. |
| 400 | `BAD_ORDER_TYPE` | 6 | `orderType` not `FULL_OPEN`; nonzero `zone`; nonzero `zoneHash`; nonzero `conduitKey`. |
| 400 | `BAD_TIME_WINDOW` | 7 | `startTime >= endTime`; start more than 300 seconds ahead; already ended; under one hour left; over ninety days left. |
| 400 | `STALE_COUNTER` | 8 | Seaport's `getCounter(offerer)` differs from the signed `counter`. |
| 400 | `NOT_OWNER` | 9 | `ownerOf(tokenId)` is not the offerer. |
| 400 | `NAME_MISMATCH` | 9 | No `tbl_domain_detail_bc` row for this token and registry, or its `domainName` differs from the submitted one. |
| 400 | `NOT_APPROVED` | 10 | `isApprovedForAll(offerer, seaport)` is false. |
| 503 | `CHAIN_UNAVAILABLE` | 8 to 10 | Any of the three reads failed after retries, and none of the others produced a reject. |
| 400 | `ALREADY_LISTED` | 11 | Another active, unexpired order exists for this token, found either by the read or by the partial unique index during insert. |
| 400 | `DUPLICATE_ORDER` | 12 | This order hash is already stored, in any status. |
| 500 | none | any | A non `OrderError` from `computeSplit`, any other database error, an unexpected throw inside `computeDigest`, or a `findByTokenId` database failure (that lookup is not wrapped in the discriminated result). |

A frontend should treat `STALE_COUNTER`, `NOT_OWNER`, `NAME_MISMATCH` and `NOT_APPROVED` as "the world changed, rebuild and re sign," `NOT_APPROVED` specifically as "send the `setApprovalForAll` transaction first," `NAME_MISMATCH` as "sync your domains," `BAD_TIME_WINDOW` as "your clock is wrong or you picked an invalid duration," `CHAIN_UNAVAILABLE` as "retry later," and every other `400` as a frontend bug.

## The diagnostic twins of the create route

Two `_health` routes expose the earlier layers over HTTP without persisting anything, so QA could exercise each sprint before the next existed. `POST /marketplacev2/orders/_health/validate-local` calls `validateLocal` (checks 1 to 7) and `POST /marketplacev2/orders/_health/validate-chain` calls `validateChain` (checks 1 to 10). Both take the same `CreateOrderDto` and both require `AccessTokenGuard`, `AdminTokenGuard` and `RequireVerifiedWalletGuard`. `AdminTokenGuard` (`src/@core/common/guards/admin-token.guard.ts`) is not a JWT check; it compares an `admin-token` request header to the `ADMIN_TOKEN` config value with loose equality (`==`). These are handy for a developer who wants to see exactly which check an order fails without creating a row, provided they have the admin token.

## The specs for this path

`order/order.service.validate-local.spec.ts` builds an `OrderService` with empty mocks for everything except the logger and config, signs real orders with `Wallet.createRandom()` and `wallet.signingKey.sign(digest).serialized`, and asserts reject codes through a small `expectRejectCode` helper. Its twelve tests: a valid order resolves with seller `97500000`, fee `2500000` and a 64 hex hash; check 1 rejects when another wallet calls; check 2 rejects after flipping one hex digit at index 10 of the signature (inside `r`, chosen because mutating `v` is not a reliable way to break recovery, as the comment explains); check 3 rejects an offer token equal to the USDT address; check 4 rejects a single consideration leg; check 5 rejects a 90/10 split; check 6 rejects `PARTIAL_OPEN`; and five check 7 tests, rejecting an expired window, accepting a start 60 seconds ahead, rejecting a start one hour ahead, rejecting a window with 30 minutes left, and rejecting a window 100 days out.

`order/order.service.validate-chain.spec.ts` replaces `seaportContract` and `domainNftContract` on the constructed service with Jest mock objects (the "override after construction" pattern, which avoids needing a real RPC). Its eight tests: the happy path, which also asserts `getCounter` was called with the wallet, `ownerOf` with `42n`, `isApprovedForAll` with the wallet and the Seaport address, and `findByTokenId` with `'42'` and the domain NFT address; local failure means no chain or database call at all; `STALE_COUNTER` with counter `5n`; `NOT_OWNER`; `NAME_MISMATCH` for a different name; `NAME_MISMATCH` for a `null` record; `NOT_APPROVED`; and `503` `CHAIN_UNAVAILABLE` when `getCounter` rejects with a plain error, which, as its comment notes, is not retryable and so fails immediately rather than waiting through real backoff delays.

`order/order.service.create.spec.ts` adds a mocked repository whose `manager.transaction` simply invokes the callback with a mock manager, and a mock query builder chain for `expireIfStale` that returns "nothing was stale." Its five tests: a valid order is inserted exactly once with `status` `active`, `maker` the wallet, `tokenContract` the NFT address, `tokenId` `'42'`, `domainName` `example.eth` and `tld` `eth`, and `upsertListed` and `record` are called with the user id and an event type `LISTED` with `txHash` `null`; check 11 rejects when `findOne` returns an existing order and asserts `insert` was never called; check 12 maps `{ code: '23505' }` to `DUPLICATE_ORDER`; a non unique database error (`'connection terminated'`) is rethrown unchanged; and `findByHash` returns the found entity.

`order/order.parity.spec.ts` (described in file `03`) runs every one of the twelve rejections plus the happy path and the `503` through `create()` end to end.

What none of them cover: the lost race path, where the insert fails with constraint `idx_marketplacev2_orders_active_listing_unique` and should become `ALREADY_LISTED` rather than `DUPLICATE_ORDER`. Every `23505` in the specs has no `constraint` property. That branch is the one that exists specifically for production concurrency, and it is untested.

## Bugs, risks and inconsistencies

A burnt or never minted token returns `503` instead of a rejection (`order.service.ts:357` and `is-retryable-chain-error.util.ts:13`). OpenZeppelin's `ownerOf` reverts for a nonexistent token, ethers reports that as `CALL_EXCEPTION`, the predicate correctly does not retry it, and `checkOwnershipAndName` then classifies it as `kind: 'unavailable'`. If the counter and approval reads succeed, the user is told "Chain temporarily unavailable. Please try again shortly," which no amount of retrying will fix. A wrong `DOMAIN_NFT_ADDRESS_POLYGON_UD` (no contract deployed) would similarly surface as `BAD_DATA` and a permanent `503`. Mapping `CALL_EXCEPTION` from `ownerOf` to `NOT_OWNER` would be more honest.

The predicate retries every ethers `SERVER_ERROR` (`is-retryable-chain-error.util.ts:13`), and ethers v6 uses that code for any non 2xx HTTP response, including `401` and `403` from an RPC provider rejecting a bad or expired API key. A revoked RPC key would make every create request wait the full four to five and a half seconds of backoff before returning `503`.

The tld is wrong for subdomains (`order.service.ts:655` with `src/@core/utils/helper.ts:57`). `splitDomainNameAndTLD` returns `res[1]`, the second label, not the last one, so `pay.alice.crypto` is stored with `tld` `alice`. Browse's `tld` filter would then never find it under `crypto`. Domains with no dot get `tld` `''` (the function returns `null` and an error which `create()` ignores), though check 9 makes that unlikely to reach this line.

`NAME_MISMATCH` is case sensitive (`order.service.ts:371`). If `tbl_domain_detail_bc` stores `Alice.crypto` and the frontend sends `alice.crypto`, the seller is told the token does not correspond to their own domain. Whether this bites depends on whether every writer to that table lowercases names, which is not enforced here.

`findByTokenId` is not inside the discriminated result (`order.service.ts:370`). A database error there escapes `checkOwnershipAndName` as a raw throw, so `Promise.all` rejects with it and the client gets a `500`, bypassing the careful reject versus unavailable logic built around it.

The lost race branch is untested (`order.service.ts:720`). The partial unique index is the only real protection against two concurrent listings of the same token, and the code that translates its violation is covered by no spec, while the index itself only exists if the deploy time autogenerated migration creates it (file `03`).

A conditional 500 on string enum names in the DTO (`order-parameters.dto.ts:25`, `offer-item.dto.ts:6`). `@IsIn(Object.values(ItemType))` is only safe if the package's `ItemType` and `OrderType` are plain objects of numbers, as the frontend mirror shows. If they are TypeScript numeric enums, `Object.values` also contains the names, so `"itemType": "ERC721"` would pass validation and then reach `computeDigest` (`order.service.ts:190`) outside any `try`, where encoding a string as `uint8` throws and becomes a `500`. Since the package source was not available for review, this is worth a two minute check.

The check numbering in the parity suite counts twelve "rejection cases," but the `503` and the guard's `WALLET_NOT_VERIFIED` are equally part of the client contract and are documented only in comments. A generated OpenAPI description would help the frontend; `@ApiBearerAuth` decorators are present but the DTOs have no `@ApiProperty` annotations.

`AdminTokenGuard` uses `==` against `this.configService.get('ADMIN_TOKEN')` (`admin-token.guard.ts:58`). If `ADMIN_TOKEN` were ever absent from config, `undefined == undefined` is true and any request without an `admin-token` header would pass. The Joi env schema lists `ADMIN_TOKEN` as required, which is the only thing preventing that.

## Appendix: every route on the order controller

The controller is mounted at `marketplacev2/orders` under the global `api/v1` prefix. Route order in the file matters: Express matches routes in declaration order, so every literal path is declared before the two `:orderHash` routes, and those use the regex constraint `ORDER_HASH_ROUTE_PATTERN = '0x[0-9a-fA-F]{64}'` from `constants/order-route.constants.ts` so that a literal segment like `my-domains` can never be swallowed by a catch all parameter and a malformed hash 404s at the router. The comment on that constant says it "structurally prevents a literal route segment ... from ever being shadowed."

| # | Method | Path under `/api/v1/marketplacev2/orders` | Guards | Service call | Notes |
|---|---|---|---|---|---|
| 1 | GET | `/_health/wallet-verified` | Access, Admin, VerifiedWallet | none | Returns `{ walletAddress }`. The `B-02` throwaway proof of the wallet guard. |
| 2 | POST | `/_health/validate-local` | Access, Admin, VerifiedWallet | `validateLocal` | Checks 1 to 7, no persistence. |
| 3 | POST | `/_health/validate-chain` | Access, Admin, VerifiedWallet | `validateChain` | Checks 1 to 10, no persistence. |
| 4 | GET | `/_health` | Access, Admin | `health` | `{ module: 'marketplacev2', status: 'ok' }`. |
| 5 | GET | `/_health/entity` | Access, Admin | `entityHealth` | Column count and index list from TypeORM metadata. |
| 6 | GET | `/_health/schema` | Access, Admin | `schemaHealth` | Live table columns and indexes via a query runner. |
| 7 | GET | `/_health/chain-config` | Access, Admin | `chainConfigHealth` | Chain id, Seaport, USDT, NFT, fee recipient, fee bps; never the RPC URL. |
| 8 | GET | `/_health/active` | Access, Admin | `activeListingsHealth` | Up to 50 newest `active` rows; status only, no `endTime` filter. Diagnostic. |
| 9 | GET | `/` | none, `Cache-Control: public, max-age=60` | `browse` | Public servable listings with search, filters, sort, category tabs, cursor pagination. |
| 10 | GET | `/stats` | none, same cache header | `stats` | Live count, sale volume, sales count, median ask, poller lag. |
| 11 | GET | `/transactions/recent` | none, same cache header | `TransactionService.findRecent` | `limit` default 12, `offset` default 0. |
| 12 | GET | `/transactions/mine` | Access, VerifiedWallet | `TransactionService.findByBuyer` | `page` default 1, `limit` default 10, via `paginateResponse`. |
| 13 | GET | `/transactions/:orderHash` (regex) | none | `TransactionService.findByOrderHash` | `404` "No transaction recorded for order ..." when none. |
| 14 | POST | `/` | Access, VerifiedWallet | `create` | This file. Returns `OrderLocalValidationResult`. |
| 15 | GET | `/my-domains` | Access | `myDomains` | Caller's owned domains with `LISTED`, `UNLISTED` or `SOLD`. |
| 16 | GET | `/my-listings` | Access, VerifiedWallet | `myListings` | Caller's orders, any status, with category filter. |
| 17 | GET | `/my-purchases` | Access, VerifiedWallet | `myPurchases` | Orders this wallet bought. |
| 18 | GET | `/listing-history` | Access, VerifiedWallet | `listingHistory` | Caller's history rows, optional `domainName` and `tokenId`. |
| 19 | POST | `/listing-status/sync` | Access, plus `apiCallLimiter` from `main.ts:85` | `ListingStatusSyncService.sync` | Refreshes ownership and corrects stale `LISTED` rows. |
| 20 | POST | `/watchlist` | Access | `WatchlistService.add` | Idempotent. |
| 21 | DELETE | `/watchlist/:id` | Access | `WatchlistService.remove` | `404` for both missing and someone else's row. |
| 22 | GET | `/watchlist` | Access | `WatchlistService.list` | Each row joined to the latest order for that token. |
| 23 | POST | `/:orderHash/cancel` (regex) | Access, VerifiedWallet | `cancel` | Soft, off chain cancel. See file `05`. |
| 24 | GET | `/:orderHash` (regex) | none | `findByHash` | Full entity; `signature` blanked to `''` unless servable. See file `05`. |

In the guards column, "Access" is `AccessTokenGuard`, "Admin" is `AdminTokenGuard` (the `admin-token` header), and "VerifiedWallet" is `RequireVerifiedWalletGuard`. The read routes (9 to 13, 15 to 18, 20 to 22) are explained in depth elsewhere in this cluster. One small inconsistency: the comment above routes 4 to 7 describes "AccessTokenGuard (not RequireVerifiedWalletGuard ...) on all four," but the code also applies `AdminTokenGuard`, so these are effectively admin only.

## Frontend note

The big lesson from this endpoint is the order of operations, and it maps cleanly onto how you would build the listing UI. Do the cheap, local things first (are you who you say you are, did you sign what you sent, does it follow our rules), then the expensive remote things in parallel (does the chain agree), then the stateful things last and atomically (does the database have room for it). And keep "you made a mistake" (`400` with a code) strictly separate from "we had a problem" (`503`), because the user's next action is completely different. If you build your listing modal to switch on `code`, you can give a precise, helpful next step for every single failure in the table above.
