# 05. Deployment Verification, the Shared Trust Boundary

## One service, deliberately shared by two features

`blockchain-deployment` is a small folder, one module and one service, `DeploymentVerificationService`. Its own doc comment states its purpose and its scope plainly, it is a "single verification engine shared by NFT collection and ERC-20 contract deployment, both need the exact same checks against a generic contract-creation transaction, so there is no reason to maintain two hand-copied services." That single sentence is worth taking seriously as an engineering principle, when two features need the exact same trust decision, write that decision once, and make both features depend on it, rather than letting two subtly different copies of the same security logic drift apart over time.

The comment is equally careful about what this service is not, "GM's check-in verification stays separate, it decodes a contract event log against a specific registered GM contract address, a genuinely different mechanism from the generic did this tx deploy a contract check here." That is a useful distinction to hold onto, this service only ever answers one question, "did this transaction hash really create a new contract, on this chain, for this user, matching what we prepared." It has no opinion about what that contract does afterward.

## Why this function is the real trust boundary of the whole cluster

Everything in file 04 ends by handing this one function four pieces of information, a `txHash` the frontend claims came from the user's own wallet, a `chainIdentifier`, the `userId` making the confirm request, and the `preparedCalldataHash` stored back at prepare time. Nothing about a `txHash` is inherently trustworthy, a user could type in a random, unrelated, real transaction hash and hope the backend accepts it. This function exists specifically to make that impossible, by running seven checks, in a fixed order, each one closing off a different way a caller could try to fake a deployment.

## Step by step, the seven checks

Step one resolves the chain and its RPC endpoint from AWS Secrets Manager, and step two actually fetches the transaction from that chain:

```ts
for (const rpcUrl of rpcUrls) {
    const candidate = new ethers.providers.JsonRpcProvider(rpcUrl);
    const candidateTx = await candidate.getTransaction(txHash);
    if (!candidateTx) { continue; }
    tx = candidateTx; provider = candidate; break;
}
```

Notice this tries every configured RPC URL for the chain in order, not just the first, so "one misbehaving endpoint doesn't block confirmation when another configured endpoint for the same chain is healthy," per the code's own comment, a small but real piece of resilience engineering. If the hash simply does not exist on that chain at all, verification fails here immediately, closing off the "type in a random hash" attempt entirely.

Step three is the one that ties this function back to a specific prepared deployment, not just any successful contract creation:

```ts
const actualCalldataHash = ethers.utils.keccak256(tx.data ?? '0x');
if (actualCalldataHash.toLowerCase() !== expectedCalldataHash.toLowerCase()) {
    return { valid: false, reason: 'Transaction calldata does not match what was prepared for this deployment.' };
}
```

Recall from file 04 that `preparedCalldataHash` was computed, at prepare time, from the exact `data` this backend itself built, bytecode plus this specific user's encoded name, symbol, supply, and owner. Hashing the real transaction's own `tx.data` and comparing it here means a caller cannot take some other, unrelated successful contract deployment (their own, or someone else's, found anywhere on chain) and use its hash to confirm a totally different pending row. The submitted transaction has to be, byte for byte, the one this backend itself prepared.

Steps four and five check that the transaction actually succeeded and actually created a contract:

```ts
if (receipt.status !== 1) {
    return { valid: false, reason: 'Transaction failed or was reverted on chain.' };
}
if (!receipt.contractAddress) {
    return { valid: false, reason: 'Transaction did not deploy a contract...' };
}
```

`receipt.status` is either `1` (succeeded) or `0` (reverted), and a reverted transaction still exists on chain and still costs the sender gas, but produces no contract and no state change, so it must never be treated as a valid deployment. `receipt.contractAddress` being present is the exact mechanic described in file 01, only present when the transaction's `to` field was empty and it genuinely created something new.

Step six confirms the transaction's sender is someone this specific user is actually allowed to deploy from:

```ts
const senderAddress = ethers.utils.getAddress(tx.from);
const registeredWallet = await this.walletRepo.findWithWalletAddrssAndUserId(senderAddress, userId);
if (!registeredWallet) {
    return { valid: false, reason: 'The transaction sender is not a wallet registered to your account.' };
}
```

Without this check, calldata matching alone would not be enough, anyone could take a legitimately prepared transaction's `data`, sign and broadcast it from an entirely different wallet they control, and the calldata hash would still match. This step is what actually ties the on chain transaction back to the authenticated user making the confirm request, not just to the deployment row.

Step seven, freshness, closes off a much longer lived kind of replay:

```ts
const txTimestampMs = block.timestamp * 1000;
const ageMs = Date.now() - txTimestampMs;
if (ageMs > MAX_TX_AGE_MS) {
    return { valid: false, reason: 'Transaction is older than 10 minutes...' };
}
```

`MAX_TX_AGE_MS` is ten minutes, defined in `nft-collection.constants.ts` and reused, per its own comment, by the ERC20 flow too. Without this, a very old, otherwise legitimate deployment transaction, one that has nothing to do with the pending row someone is trying to confirm right now, could theoretically still pass every other check if its calldata happened to collide (astronomically unlikely, but the check costs nothing and removes the possibility entirely). A tight freshness window keeps "prepare then confirm" close together in time, which matches how the feature is actually meant to be used.

## The one line of defense this function deliberately leaves to its caller

The function's own doc comment ends with an honest disclosure of a gap it does not close itself, "Duplicate-hash replay protection is enforced separately in the calling services via the txHash uniqueness constraint in the database." This function only ever asks "is this transaction hash a real, fresh, ownership matching, contract creating deployment of exactly this prepared calldata." It deliberately never checks whether that same hash has already been used to confirm some other row, because that is a database concern, not a blockchain concern, and file 04 showed exactly where that other half lives, the `findByTxHash` check and the unique constraint on the `txHash` column. Together, the two halves close the loop completely, this file guarantees a hash is genuine, the calling service guarantees it can only ever be spent once.

## Where to go next

[06-alchemy-and-the-alchmey-folder.md](06-alchemy-and-the-alchmey-folder.md) moves from writing and verifying transactions to a different kind of blockchain reading, answering "what does this wallet already own," through a third party data provider rather than a direct RPC connection.
