# 04. Contract Deployment and NFT Collection, the Prepare and Confirm Pattern

## Two features, one shared shape

`contract-deployment` lets a user deploy their own ERC20 token, using the `EndlessToken.sol` template from file 01. `nft-collection` lets a user deploy their own NFT collection, using `EndlessCollection.sol`. Reading `contract-deployment.service.ts` and `nft-collection.service.ts` side by side, they are, deliberately, the same pattern applied twice, and the code says so itself, `ContractDeploymentService.prepareContract` even comments that its rate limiting "mirrors the NFT collection prepare pattern." Both expose the same two step flow, `prepare` and `confirm`, and understanding one means understanding the other.

## Why deployment is split into two steps at all

The most important thing to notice before reading either service's code is what this backend never does, it never holds a private key capable of signing a deployment transaction on a user's behalf. Deploying a contract, as file 01 explained, is just a transaction, and every transaction needs a signature from the account paying for it. Since the deployed token or collection needs to actually belong to the user, not to Endless Domains, the user's own wallet has to be the one that signs it. That forces the flow into two separate calls, one where the backend builds everything except the signature, and one where the backend checks what the user's wallet actually did with it.

## Prepare, building an unsigned transaction

Look at the core of `ContractDeploymentService.prepareContract`:

```ts
const supplyWei = ethers.utils.parseUnits(dto.supply, OZ_ERC20_DECIMALS);

const iface = new ethers.utils.Interface(OZ_ERC20_CONSTRUCTOR_ABI);
const encodedArgs = iface.encodeDeploy([dto.name, dto.symbol, supplyWei, ownerAddress]);
const data = OZ_ERC20_BYTECODE + encodedArgs.slice(2);

const gasEstimate = await this.estimateGas(chainConfig, data, signerAddress);
const preparedCalldataHash = ethers.utils.keccak256(data);
```

`ethers.utils.parseUnits` converts a human readable number like "1000" into the token's actual smallest unit representation, scaled by `10 ** 18` since `OZ_ERC20_DECIMALS` is fixed at 18, exactly matching how the contract's own doc comment in file 01 described `initialSupply_` as "already scaled by 10**18." From there, the flow is exactly the encode and concatenate mechanic from file 02, producing `data`, the complete payload of an unsigned contract creation transaction. `estimateGas` asks a real RPC node how much gas that data would cost to execute, purely a simulation, nothing is sent. `keccak256(data)` produces a fixed length fingerprint of that exact payload, stored on the pending database row as `preparedCalldataHash`, this single value is what makes the later confirm step trustworthy, covered in full in file 05.

`NftCollectionService.prepareCollection` follows an identical shape but does noticeably more work first, because deploying a collection also means giving it real metadata:

```ts
const imageUrl = await this.uploadCoverImage(imageFile);
const metadataUri = await this.uploadMetadataToIpfs(dto.name, dto.description, imageUrl);

const symbol = this.deriveSymbol(dto.name);
const iface = new ethers.utils.Interface(OZ_ERC721_CONSTRUCTOR_ABI);
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
