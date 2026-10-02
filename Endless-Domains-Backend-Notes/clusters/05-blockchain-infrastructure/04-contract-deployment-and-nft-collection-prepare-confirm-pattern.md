# 04. Contract Deployment and NFT Collection, the Prepare and Confirm Pattern

## Two features, one shared shape

`contract-deployment` lets a user deploy their own ERC20 token, using the `EndlessToken.sol` template from file 01. `nft-collection` lets a user deploy their own NFT collection, using `EndlessCollection.sol`. Reading `contract-deployment.service.ts` and `nft-collection.service.ts` side by side, they are, deliberately, the same pattern applied twice, and the code says so itself, `ContractDeploymentService.prepareContract` even comments that its rate limiting "mirrors the NFT collection prepare pattern." Both expose the same two step flow, `prepare` and `confirm`, and understanding one means understanding the other.

## Why deployment is split into two steps at all

The most important thing to notice before reading either service's code is what this backend never does, it never holds a private key capable of signing a deployment transaction on a user's behalf. Deploying a contract, as file 01 explained, is just a transaction, and every transaction needs a signature from the account paying for it. Since the deployed token or collection needs to actually belong to the user, not to Endless Domains, the user's own wallet has to be the one that signs it. That forces the flow into two separate calls, one where the backend builds everything except the signature, and one where the backend checks what the user's wallet actually did with it.

## Prepare, building an unsigned transaction

Look at the core of `ContractDeploymentService.prepareContract`:

```ts
// src/components/contract-deployment/services/contract-deployment.service.ts (lines 61 to 68, ethers v6 form)
const supplyWei = ethers.parseUnits(dto.supply, OZ_ERC20_DECIMALS);

const iface = new ethers.Interface(OZ_ERC20_CONSTRUCTOR_ABI);
const encodedArgs = iface.encodeDeploy([dto.name, dto.symbol, supplyWei, ownerAddress]);
const data = OZ_ERC20_BYTECODE + encodedArgs.slice(2);

const gasEstimate = await this.estimateGas(chainConfig, data, signerAddress);
const preparedCalldataHash = ethers.keccak256(data);
```

`ethers.parseUnits` (spelled `ethers.utils.parseUnits` before commit `71fd2fec`, and returning a `bigint` now instead of a v5 `BigNumber`) converts a human readable number like "1000" into the token's actual smallest unit representation, scaled by `10 ** 18` since `OZ_ERC20_DECIMALS` is fixed at 18, exactly matching how the contract's own doc comment in file 01 described `initialSupply_` as "already scaled by 10**18." From there, the flow is exactly the encode and concatenate mechanic from file 02, producing `data`, the complete payload of an unsigned contract creation transaction. `estimateGas` asks a real RPC node how much gas that data would cost to execute, purely a simulation, nothing is sent. `keccak256(data)` produces a fixed length fingerprint of that exact payload, stored on the pending database row as `preparedCalldataHash`, this single value is what makes the later confirm step trustworthy, covered in full in file 05.

`NftCollectionService.prepareCollection` follows an identical shape but does noticeably more work first, because deploying a collection also means giving it real metadata:

```ts
// src/components/nft-collection/services/nft-collection.service.ts (lines 63 to 69, ethers v6 form)
const imageUrl = await this.uploadCoverImage(imageFile);
const metadataUri = await this.uploadMetadataToIpfs(dto.name, dto.description, imageUrl);

const symbol = this.deriveSymbol(dto.name);
const iface = new ethers.Interface(OZ_ERC721_CONSTRUCTOR_ABI);
const encodedArgs = iface.encodeDeploy([dto.name, symbol, metadataUri, ownerAddress]);
const data = OZ_ERC721_BYTECODE + encodedArgs.slice(2);
```

The cover image the user uploads goes to S3 first (`uploadCoverImage`), producing a plain HTTPS URL, and then a small JSON object, `{ name, description, image }`, gets pinned to IPFS through Pinata (`uploadMetadataToIpfs`), producing an `ipfs://` URI. That URI becomes `baseTokenURI_`, the single metadata address every token in the collection will share, exactly the quirk documented on the contract itself in file 01. `deriveSymbol` is a small, honest piece of logic worth reading once, it upper cases the collection's name, strips anything that is not a letter or digit, and takes the first five characters, falling back to the literal string `NFT` if nothing survives that filter, a sensible default for a field the contract requires but the user never actually types themselves.

Both `prepare` methods end the same way, they persist a database row in `PENDING` status holding the unsigned `data`, the `preparedCalldataHash`, and a `configSnapshot` (or the collection's own fields), and hand the caller back exactly what a frontend needs to actually get a signature, an `unsignedTx` object carrying `data`, `chainId`, and `gasEstimate`. From this point forward, it is the frontend's job to pass that `data` to the user's connected wallet (something like MetaMask), have the wallet actually sign and broadcast it, and get back a real transaction hash.

## Confirm, checking what actually happened on chain

Once the frontend has a `txHash` from the user's wallet, it calls the matching confirm endpoint, and both services again follow the identical shape, seen clearly in `ContractDeploymentService.confirmContract`:

```ts
const deployment = await this.contractDeploymentRepository.findById(deploymentId);
if (deployment.userId !== userId) {
    throw new ForbiddenException('This contract deployment does not belong to your account.');
}
if (deployment.deploymentStatus === ContractDeploymentStatus.CONFIRMED) {
    return this.buildConfirmResponse(deployment, 'This contract deployment has already been confirmed.');
}
...
await this.contractDeploymentRepository.markSubmitted(deployment.id);

const result = await this.deploymentVerificationService.verifyDeploymentTransaction(
    txHash, deployment.chainId, userId, deployment.preparedCalldataHash
);
if (!result.valid) {
    await this.contractDeploymentRepository.markFailed(deployment.id);
    throw new BadRequestException(result.reason ?? 'On-chain deployment verification failed.');
}

confirmed = await this.contractDeploymentRepository.confirmDeployment(deployment.id, txHash, result.contractAddress);
```

Before any blockchain call happens at all, ownership is checked (a caller can only confirm their own pending row), the row's current status is checked (an already confirmed row just returns its own data again rather than re-verifying, a deliberately idempotent retry path), and a rate limit is checked and consumed, because, as the comment on `CONFIRM_RATE_LIMIT` puts it plainly, "each attempt triggers a real RPC verification call," a real, metered cost this company pays per attempt. Only after all of that does the actual verification happen, entirely delegated to `DeploymentVerificationService.verifyDeploymentTransaction`, the single shared engine covered in full in file 05. If it comes back invalid, the row is marked `FAILED` and a `BadRequestException` carries the specific reason back to the caller. If it comes back valid, the row flips to `CONFIRMED`, permanently storing the real, on chain `contractAddress` the deployment produced.

One more detail worth internalizing, because it shows a team thinking about abuse rather than just the happy path, both services also handle a duplicate `txHash` explicitly, before and after the database write:

```ts
const existingByTxHash = await this.contractDeploymentRepository.findByTxHash(txHash);
if (existingByTxHash) {
    throw new ConflictException('This transaction has already been used to confirm a different deployment.');
}
```

and, on the write itself, a caught unique constraint violation (`err?.code === '23505'`) covers the narrow race where two confirm requests for the same row land at nearly the same instant. A `txHash` can only ever confirm the one deployment it was actually produced for, never be replayed to make a second, unrelated pending row look confirmed.

## Why both flows freeze during a cron run

Both `confirmContract` and `confirmDeployment` open with the same check, worth noticing precisely because it has nothing to do with blockchains at all:

```ts
const cronActive = await this.systemFlagRepo.getCronRunning();
if (cronActive) {
    throw new HttpException('Scores are being updated...', HttpStatus.SERVICE_UNAVAILABLE);
}
```

This ties deployment confirmation to a shared, application wide flag that a separate reputation scoring cron job sets while it recalculates user scores, deliberately borrowed, per the comment, from "the perk-claiming convention." It is a small reminder that this cluster's code does not live in isolation, contract and collection deployments feed into a wider reputation and rewards system covered elsewhere in this project's notes, and this freeze exists to keep that system's numbers consistent while it is mid recalculation.

## Where to go next

[05-deployment-verification-the-shared-trust-boundary.md](05-deployment-verification-the-shared-trust-boundary.md) is a full, step by step read of `verifyDeploymentTransaction`, the one function both of these confirm flows lean on entirely to decide whether a submitted transaction hash can be trusted.

## Update from the October 2026 uat pull

Both services were migrated from ethers v5 to ethers v6 in commit `71fd2fec` ("Ether versoin 6"), and the two prepare excerpts above have been corrected in place. In `src/components/contract-deployment/services/contract-deployment.service.ts` the changes are line 43 and line 51 (`ethers.utils.getAddress` to `ethers.getAddress` for the signer and the optional owner), line 61 (`ethers.utils.parseUnits` to `ethers.parseUnits`), line 63 (`new ethers.utils.Interface` to `new ethers.Interface`), line 68 (`ethers.utils.keccak256` to `ethers.keccak256`) and lines 278 to 280 inside `estimateGas`. `src/components/nft-collection/services/nft-collection.service.ts` got the mirror image at lines 58, 67, 72 and 313 to 315. Every one of these is a rename, the encoding and hashing algorithms are byte for byte identical between the two major versions.

The one change that is not a pure rename is the gas estimate conversion inside `estimateGas`:

```ts
// src/components/nft-collection/services/nft-collection.service.ts (lines 313 to 315)
const provider = new ethers.JsonRpcProvider(rpcUrl);
const estimate = await provider.estimateGas({ data, from: fromAddress });
return Number(estimate);
```

In v5, `provider.estimateGas` returned a `BigNumber` and the old code called `estimate.toNumber()`. In v6 it returns a native `bigint`, which has no `.toNumber()` method, so the old line would have thrown a `TypeError` that the surrounding `catch` silently turns into `DEFAULT_GAS_ESTIMATE_FALLBACK`. Nothing would have looked broken, every prepare call would just have returned the fallback gas number. `Number(estimate)` is the correct v6 replacement and is safe because gas values are nowhere near `Number.MAX_SAFE_INTEGER`. It also matters for the response: `gasEstimate` ends up inside the JSON body returned to the frontend (`unsignedTx.gasEstimate` and `gasEstimate`), and `JSON.stringify` throws "Do not know how to serialize a BigInt" if handed a raw `bigint`, so converting before returning is not optional in v6. Similarly, `supplyWei` from `ethers.parseUnits` is now a `bigint`, but it is only ever passed into `encodeDeploy` and never stored or returned, while the persisted `configSnapshot.supply` stays the original string `dto.supply` (line 73), so there is no serialization risk there.

This pull also added the first spec files for both services. `src/components/contract-deployment/services/contract-deployment.service.spec.ts` (4 tests) and `src/components/nft-collection/services/nft-collection.service.spec.ts` (4 tests) share one structure. They replace `ethers.JsonRpcProvider` with a `jest.fn()` using `jest.mock('ethers', ...)` while keeping every other real ethers function through `jest.requireActual`, build the service by hand with plain object mocks for the repository, secrets service, logger, S3 and Pinata, and then test three things: that `estimateGas` turns a mocked `2_100_000n` (or `1_850_000n`) into the plain number `2100000` with `typeof result` equal to `'number'`, that a rejected `estimateGas` returns the fallback, that a malformed `walletAddress` throws `BadRequestException` before any RPC, S3 or IPFS call, and that a fixed set of inputs produces a pinned `preparedCalldataHash` (`0xe356...acfdb` at line 109 of the ERC20 spec, `0xfc6b...7343d` at line 119 of the NFT spec). One honest caveat on that last test: the test titles say "matching the pre migration v5 output", but the comments directly underneath say the hash was "computed once under v6 and pinned here". So the test proves the hash will not drift in the future, it does not by itself prove that v6 produces exactly what v5 produced. In practice they do agree, because keccak256 and ABI encoding did not change between versions, but the evidence for that is the ethers changelog, not this test.

Two smaller leftovers are worth knowing about. The doc comments at `src/components/contract-deployment/constants/oz-erc20-template.constant.ts` line 5 and `src/components/nft-collection/constants/oz-erc721-template.constant.ts` line 5 still say "This is all `ethers.utils.Interface.encodeDeploy` needs", and the generator scripts `src/scripts/write-erc20-template-constant.js` and `src/scripts/write-nft-template-constant.js` line 16 still write that same v5 wording into the constant files whenever they are regenerated. That is only stale documentation, nothing executes it, but a reader copying the name will hit "Cannot read properties of undefined (reading 'Interface')" under v6. Second, the per request `new ethers.JsonRpcProvider(rpcUrl)` at line 278 (ERC20) and line 313 (NFT) has no `staticNetwork` option, so a dead RPC URL can make the prepare call stall rather than fall back quickly, the same v6 behaviour explained in [05-deployment-verification-the-shared-trust-boundary.md](05-deployment-verification-the-shared-trust-boundary.md). The reputation cron that sets the `getCronRunning` freeze flag lives in `src/components/reputation-gm-perk/reputation/services/reputation.service.ts` line 408, not in the `CronModule` that this pull commented out, so the freeze described above still works exactly as before. The codebase wide migration story is in [../12-marketplace-v2/12-ethers-v6-migration-and-the-new-test-suite.md](../12-marketplace-v2/12-ethers-v6-migration-and-the-new-test-suite.md).
