# 05. Deployment Verification, the Shared Trust Boundary

## One service, deliberately shared by two features

`blockchain-deployment` is a small folder, one module and one service, `DeploymentVerificationService`. Its own doc comment states its purpose and its scope plainly, it is a "single verification engine shared by NFT collection and ERC-20 contract deployment, both need the exact same checks against a generic contract-creation transaction, so there is no reason to maintain two hand-copied services." That single sentence is worth taking seriously as an engineering principle, when two features need the exact same trust decision, write that decision once, and make both features depend on it, rather than letting two subtly different copies of the same security logic drift apart over time.

The comment is equally careful about what this service is not, "GM's check-in verification stays separate, it decodes a contract event log against a specific registered GM contract address, a genuinely different mechanism from the generic did this tx deploy a contract check here." That is a useful distinction to hold onto, this service only ever answers one question, "did this transaction hash really create a new contract, on this chain, for this user, matching what we prepared." It has no opinion about what that contract does afterward.

## Why this function is the real trust boundary of the whole cluster

Everything in file 04 ends by handing this one function four pieces of information, a `txHash` the frontend claims came from the user's own wallet, a `chainIdentifier`, the `userId` making the confirm request, and the `preparedCalldataHash` stored back at prepare time. Nothing about a `txHash` is inherently trustworthy, a user could type in a random, unrelated, real transaction hash and hope the backend accepts it. This function exists specifically to make that impossible, by running seven checks, in a fixed order, each one closing off a different way a caller could try to fake a deployment.

## Step by step, the seven checks

Step one resolves the chain and its RPC endpoint from AWS Secrets Manager, and step two actually fetches the transaction from that chain:

```ts
// src/components/blockchain-deployment/services/deployment-verification.service.ts (lines 70 to 86, simplified, ethers v6 form)
for (const rpcUrl of rpcUrls) {
    const candidate = new ethers.JsonRpcProvider(rpcUrl);
    const candidateTx = await candidate.getTransaction(txHash);
    if (!candidateTx) { continue; }
    tx = candidateTx; provider = candidate; break;
}
```

Notice this tries every configured RPC URL for the chain in order, not just the first, so "one misbehaving endpoint doesn't block confirmation when another configured endpoint for the same chain is healthy," per the code's own comment, a small but real piece of resilience engineering. If the hash simply does not exist on that chain at all, verification fails here immediately, closing off the "type in a random hash" attempt entirely.

Step three is the one that ties this function back to a specific prepared deployment, not just any successful contract creation:

```ts
// src/components/blockchain-deployment/services/deployment-verification.service.ts (lines 103 to 107)
const actualCalldataHash = ethers.keccak256(tx.data ?? '0x');
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
// src/components/blockchain-deployment/services/deployment-verification.service.ts (lines 140 to 155, simplified)
const senderAddress = ethers.getAddress(tx.from);
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

## Update from the October 2026 uat pull

The logic of the seven checks did not change at all in this pull, but every ethers call inside them was rewritten by commit `71fd2fec` ("Ether versoin 6") for the ethers v5 to v6 upgrade, and the excerpts above have been corrected in place to match. The full diff for `src/components/blockchain-deployment/services/deployment-verification.service.ts` is seven lines. Lines 67 and 68 changed the variable types from `ethers.providers.JsonRpcProvider` and `ethers.providers.TransactionResponse` to `ethers.JsonRpcProvider` and `ethers.TransactionResponse`. Line 71 changed `new ethers.providers.JsonRpcProvider(rpcUrl)` to `new ethers.JsonRpcProvider(rpcUrl)`. Line 103 changed `ethers.utils.keccak256` to `ethers.keccak256`. Line 110 changed `ethers.providers.TransactionReceipt` to `ethers.TransactionReceipt`. Line 142 changed `ethers.utils.getAddress` to `ethers.getAddress`. Line 158 changed `ethers.providers.Block` to `ethers.Block`. Those are pure renames, v6 simply removed the `utils` and `providers` namespaces and exported everything at the top level, so the hashing, the checksum normalisation and the receipt shape behave identically. `receipt.status` is still a plain number (`1` for success, `0` for revert, typed `number | null` in v6) and `block.timestamp` is still a plain number of seconds, so `receipt.status !== 1` at line 125 and `block.timestamp * 1000` at line 166 needed no change and do not mix `bigint` with `number`.

The pull also added the first real test file for this service, `src/components/blockchain-deployment/services/deployment-verification.service.spec.ts` (251 lines, 17 tests). It replaces `ethers.JsonRpcProvider` with a `jest.fn()` through `jest.mock('ethers', ...)` at lines 5 to 14, overrides `ethers.keccak256` in a `beforeEach` at line 67 so the calldata check is controlled by the test rather than by real bytes, and builds fake provider objects with `makeProvider()` at lines 40 to 46. There is one test per failure path of each of the seven steps (unsupported chain, missing RPC, transaction not found, fetch failing on every RPC, missing `preparedCalldataHash`, calldata mismatch, receipt fetch failure, pending receipt, reverted status, no contract address, unparseable sender, unregistered sender, block fetch failure, stale block), one happy path test, a failover test at lines 210 to 234 proving the second RPC is used once the first rejects, and a `block.timestamp` arithmetic test at lines 238 to 250. Be honest with yourself about that last one: the mock itself returns a number, so asserting that the timestamp is a number proves the arithmetic does not throw, but it cannot prove what a real v6 provider returns. The real guarantee comes from the ethers v6 type definitions, not from this test.

Two v6 specific risks are worth writing down, because neither is covered by those tests.

The first is the provider construction at line 71. In v6 a `JsonRpcProvider` built without a network hint detects the chain lazily, and when the node is unreachable it logs "JsonRpcProvider failed to detect network and cannot start up; retry in 1s" and keeps retrying in the background. The marketplace v2 code in this same repository explicitly guards against this by passing `{ staticNetwork: true }` and a chain id, and its comment at `src/components/marketplacev2/order/order.service.ts` lines 154 to 159 states the consequence plainly, "without it, a down/unreachable RPC makes the provider retry network detection every second forever." This service does not pass that option. The concrete scenario: `POL_DEPLOY_RPC` points at a dead host, a user confirms a deployment, and instead of the first candidate failing quickly so the loop moves on to `POL_GM_RPC` (the whole point of the failover written at lines 70 to 86), the request may stall until the HTTP layer gives up, and a new retrying provider object is left behind for every confirm attempt. Because the tests mock `JsonRpcProvider` entirely, this path is never exercised. The fix is a one line change to `new ethers.JsonRpcProvider(rpcUrl, Number(chainConfig.chainId), { staticNetwork: true })` and is worth confirming against a dead URL locally.

The second is that v6 types `provider.getBlock()` as returning `Block | null`. This repository's `tsconfig.json` does not enable `strict` or `strictNullChecks`, so `let block: ethers.Block` at line 158 compiles even though the real return type allows `null`. If a load balanced RPC answers the receipt call from one node and the block call from a node that has not yet seen that block, `getBlock` resolves to `null`, the `try` at lines 159 to 164 does not catch anything because nothing threw, and `block.timestamp` at line 166 throws `TypeError: Cannot read properties of null`, which escapes the function and turns into a 500 response instead of a clean `{ valid: false }`. The same pattern exists in the GM verifier, covered in [../08-affiliate-and-loyalty/04-gm-daily-checkin-and-streaks.md](../08-affiliate-and-loyalty/04-gm-daily-checkin-and-streaks.md). A one line `if (!block) return { valid: false, reason: 'Failed to fetch block details for transaction.' };` would close it.

For how the newer marketplace v2 code reads the chain with a single long lived provider and a retry predicate, see [../12-marketplace-v2/08-the-onchain-event-poller-architecture.md](../12-marketplace-v2/08-the-onchain-event-poller-architecture.md), and for the codebase wide migration story see [../12-marketplace-v2/12-ethers-v6-migration-and-the-new-test-suite.md](../12-marketplace-v2/12-ethers-v6-migration-and-the-new-test-suite.md).
