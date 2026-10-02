# 04. When the integration writes to a chain instead of asking someone else's API

## Why these three folders sit apart from everything else in this cluster

Every folder covered so far, no matter how big or how small, ultimately does the same kind of thing: it asks somebody else's server a question over HTTP and reads back an answer. `ens-integration`, `eth-domain-renewal`, and `bnb-arb-domain-renewal` are different in kind, not just in size. None of them calls a registrar's REST API at all. All three talk directly to a blockchain node using `web3` and `ethers`, reading and writing smart contract state rather than someone else's web server. That single fact is what makes them worth reading together, and the difference in who actually signs the transaction between `ens-integration` and the two renewal folders is the most important architectural decision in this entire cluster.

## ens-integration: the company's own wallet does the signing

`EnsIntegrationService` is how a `.eth` domain actually gets registered onto Ethereum once a buyer has paid. ENS's own protocol requires a commit and reveal sequence to stop front running: you first submit a hash committing to the name and a secret, wait long enough that nobody watching the chain can beat you to registering it, then reveal the real registration. `transferDomain` walks exactly that sequence:

```ts
const secrets: any[] = [];
while (i < domainNames.length) { secrets.push(await this.generateRandomSecret()); i++; }
const ensBulkRegisterContract = await this.getENSBulkRegisterContract();
const commitResponse = await this.commit(domainNames, owners, secrets, ensBulkRegisterContract);
await new Promise((r) => setTimeout(r, 80000));
const registerResponse = await this.register(domainNames, owners, secrets, durations, ensBulkRegisterContract);
```

The eighty second `setTimeout` sitting in the middle of a request handling method is a striking detail on its own: this is a synchronous wait inside a live HTTP call, meaning whatever endpoint calls `transferDomain` is going to sit open for at least eighty seconds before it can return, entirely because ENS's own protocol requires a delay between commit and reveal. Both `commit` and `register`, the two private methods either side of that wait, build their transaction the same way: encode the contract call, estimate gas, pad the estimate with a seventy five percent multiplier as a safety margin, sign it with a private key read straight out of config (`this.WALLET_PRIVATE_KEY`), and broadcast it with `web3.eth.sendSignedTransaction`. That private key belongs to the company, not the buyer. The company's own wallet pays the gas and holds custody of the registration transaction from start to finish, on the buyer's behalf.

Two smaller methods round the file out. `estimateFees` runs the same gas estimate and rent price lookup as the real registration, but against freshly generated random addresses instead of real ones, purely to quote a buyer a price before they commit to paying, and falls back to a hardcoded `0.0067` ETH and a hardcoded `2221.79` USD per ETH rate if either the live gas estimate or the CoinGecko exchange rate call fails, both marked with a `TODO` to eventually pull a real fallback out of the database instead. `getGasPrice` tries Etherscan's gas oracle first and only falls back to the node's own `web3.eth.getGasPrice()`, multiplied by 1.2 for margin, if Etherscan itself fails.

## eth-domain-renewal and bnb-arb-domain-renewal: the buyer's own wallet signs instead

The two renewal folders solve a related problem, letting a domain owner extend how long they own a name, but they make the opposite custody decision on purpose, and the code says so directly: `auditLog` in both services carries the comment "this feature is non custodial and never holds either," referring to a private key or a signature. Instead of the company signing anything, the flow is split into three steps that map directly onto the three controller routes each one exposes. `getRenewalStatus` and `getRenewalQuote` are read only, checking a name's real onchain expiry against the base registrar contract and quoting a live rent price with a gas estimate layered on top. `prepareRenewal` builds the exact calldata for the real `renew()` call, hashes it, and saves that hash to Postgres alongside the quoted price and the caller's own wallet address, then hands the unsigned transaction details straight back to whoever called it:

```ts
// src/components/eth-domain-renewal/services/eth-domain-renewal.service.ts (lines 319 to 320, ethers v6 form since commit 26b1f0e4)
const data = this.encodeRenewCalldata(quote.label, dto.durationSeconds);
const preparedCalldataHash = ethers.keccak256(data);
...
return { renewalId: record.id, to: controllerAddress, data, value: bufferedValueWei, ... chainId: CHAIN_ID_BY_NETWORK[this.blockchainNetwork], ... };
```

The frontend, not this backend, is the thing that actually asks the user's own wallet to sign that `to`/`data`/`value`/`chainId` payload and broadcast it. Only after that happens does the user call `confirmRenewal` with the resulting transaction hash, and `EthDomainRenewalVerificationService` does the actual checking: it fetches the transaction from the chain, hashes its input data, and refuses to confirm unless that hash matches the one saved at prepare time:

```ts
const dataHash = this.web3.utils.keccak256(tx.input);
if (dataHash !== record.preparedCalldataHash) {
    throw new BadRequestException('Transaction calldata does not match the prepared renewal, refusing to confirm');
}
```

That check exists specifically so a user cannot submit some unrelated transaction hash and have this backend mark their renewal confirmed anyway. The verification service also insists the transaction is recent enough (`MAX_TX_AGE_MS`, ten minutes) and checks the base registrar's own `nameExpires` after the fact to make sure the new expiry actually moved forward by at least the paid duration, and it treats a transaction that reverted on chain (`receipt.status` false) as a real, recorded `FAILED` state rather than silently ignoring it. Both renewal services also layer real defense in depth around all of this that the check heavy read only methods in the smaller chain folders never bother with: a per user rate limit on both `prepareRenewal` (five per hour) and `confirmRenewal` (ten per hour), a hard ceiling on the quoted price so a corrupted or manipulated quote can never be prepared (`RENEWAL_PRICE_CEILING_WEI`), a unique database constraint on `txHash` so the same onchain transaction can never confirm two renewal rows, and a `classifyKnownError` helper that turns a caught exception into a stable, loggable outcome string for every branch that can fail.

`bnb-arb-domain-renewal` is the same design generalized across two chains at once rather than duplicated, and the way it does that generalization is itself worth reading. `SPACE_ID_CHAIN_CONFIG`, a single exported constant, holds every per chain difference the feature cares about, RPC URL env var, contract ABI, TLD suffix, gas fallback, price ceiling, keyed by a `SpaceIdChain` enum of `BNB` and `ARB`, so every method in the service takes a `chain` parameter and looks its configuration up in that one map instead of branching on `chain === 'bnb'` by hand in a dozen different places. The service also resolves its registrar controller address live from the SID registry on every single call rather than caching it, and the comment explains exactly why: "SPACE ID rotates controllers often, this is the exact three step lookup `@web3-name-sdk/register`'s own `SIDRegister.getRegistrarController()` performs, replicated here directly with web3 rather than adding that package as a new dependency." That is a deliberate, explained tradeoff, not an accident, choosing to keep one more dependency out of the codebase at the cost of reimplementing a small piece of someone else's SDK by hand.

## The takeaway worth remembering

`ens-integration` shows what it looks like when a company custodies the transaction itself, simpler for the buyer, but it means the company's own wallet is the one paying gas and holding risk on every registration, and a synchronous eighty second wait sits inside the request. The two renewal folders show the opposite choice made carefully: more moving parts, a prepare and confirm split, a rate limiter, a calldata hash check, but the user's own wallet signs and pays for its own transaction the whole way through, and the backend's only job is to quote a price, hand back the right calldata, and later verify what actually happened on chain. Reading these three side by side is a better lesson in custodial versus self custodied Web3 backend design than almost anything else in this codebase.

## Update from the October 2026 uat pull

All three folders in this note were touched by the ethers v5 to v6 upgrade, mostly in commit `26b1f0e4` ("Upgraded the ethers version", 14 September 2026), which followed the `package.json` bump in `71fd2fec`. The prepare excerpt above has been corrected in place. None of the custody decisions described above changed, the company still signs ENS registrations with its own key through web3.js, and the renewal folders still never touch a user's key.

In `src/components/eth-domain-renewal/services/eth-domain-renewal.service.ts`, line 176 changed `new ethers.utils.Interface(ENSContract.abi)` to `new ethers.Interface(ENSContract.abi)`, line 292 changed `ethers.utils.getAddress(dto.walletAddress)` to `ethers.getAddress(dto.walletAddress)`, and line 320 changed `ethers.utils.keccak256(data)` to `ethers.keccak256(data)`. The shared helper both the renewal service and its verifier rely on changed in exactly one line:

```ts
// src/components/eth-domain-renewal/utils/label-hash.util.ts (whole file, lines 1 to 6)
import { ethers } from 'ethers';

// Shared so the service and the verification service always hash a label the same way.
export function labelHash(label: string): string {
    return ethers.keccak256(ethers.toUtf8Bytes(label));
}
```

It used to read `ethers.utils.keccak256(ethers.utils.toUtf8Bytes(label))`. `toUtf8Bytes` turns the JavaScript string into a `Uint8Array` of UTF 8 bytes, and `keccak256` hashes those bytes into a `0x` prefixed 64 character hex string, which is exactly the ENS "labelhash" and, read as a `uint256`, the token id of the name on the base registrar. This little function is now imported outside its own folder too, the BNB expiry lookup added to `src/components/alchmey/bnb-alchmey/bnb-alchmey.serveice.ts` line 19 uses it to call `nameExpires` on the SPACE ID base registrar (see [../05-blockchain-infrastructure/06-alchemy-and-the-alchmey-folder.md](../05-blockchain-infrastructure/06-alchemy-and-the-alchmey-folder.md)). The new spec `src/components/eth-domain-renewal/utils/label-hash.util.spec.ts` (two tests) pins `labelHash('example')` to `0x6fd43e7cffc31bb581d7421c8698e29aa2bd8e7186a394b85299908b4eb9b175` and checks that a different label gives a different hash, so any regression in the UTF 8 or hashing pipeline, or a reintroduced v5 import, fails loudly.

In `src/components/bnb-arb-domain-renewal/services/bnb-arb-domain-renewal.service.ts`, line 179 changed `ethers.utils.namehash(config.tld)` to `ethers.namehash(config.tld)`, line 228 changed `new ethers.utils.Interface(...)` to `new ethers.Interface(...)`, line 355 changed `ethers.utils.getAddress` to `ethers.getAddress`, and line 383 changed `ethers.utils.keccak256` to `ethers.keccak256`. `namehash` is the recursive ENS style hash of a full dotted name (here just the TLD, `bnb` or `arb`) that the SID registry uses as its key, and its algorithm is identical in both versions, but v6 normalises names with the newer ENSIP 15 rules (through the `@adraffy/ens-normalize` dependency in `package-lock.json`), so an unusual Unicode label could normalise differently than in v5. For the plain ASCII TLDs used here that cannot happen. The new spec `src/components/bnb-arb-domain-renewal/services/bnb-arb-domain-renewal.service.spec.ts` (two tests) builds the service with a fake `ConfigService` holding localhost RPC URLs, calls the private `encodeRenewCalldata(SpaceIdChain.BNB, 'testlabel', 31536000)`, and compares the result against a pinned 266 character calldata string beginning `0xacf1a841`, which is the 4 byte selector of `renew(string,uint256)`. The comment says the golden value was computed under v6, so it guards against future drift rather than proving v5 parity.

In `src/components/ens-integration/ens-integration.service.ts`, the only change is line 227, inside `estimateFees`:

```ts
// src/components/ens-integration/ens-integration.service.ts (lines 224 to 229)
while (i < domainNames.length) {
    secrets.push(await this.generateRandomSecret());
    // Generating random wallet address for estimating gas
    owners.push(ethers.Wallet.createRandom().address);
    i++;
}
```

The old line was `ethers.Wallet.createRandom(['i']).address`. A one element array was never a meaningful argument, in v5 `createRandom` took an options object and simply ignored the array, and in v6 the only accepted argument is an optional `Provider`, so the array became a compile error. The fix drops the argument, which changes nothing at runtime because the code only ever reads `.address` to get a throwaway owner for a gas estimate. One small v6 detail worth knowing: `createRandom()` now returns an `HDNodeWallet` derived from a freshly generated mnemonic rather than a plain `Wallet`, which costs a little more CPU per call (it runs the BIP 39 key derivation), and this loop runs once per domain in the quote. The existing `ens-integration.service.spec.ts` gained a new `describe` block with two tests that call `ethers.Wallet.createRandom().address`, check that `ethers.getAddress(address)` returns it unchanged (proving it is already a valid EIP 55 checksummed address), and check that two calls give two different addresses. Those tests exercise ethers itself rather than the service method, so they would still pass if someone reintroduced a bug inside `estimateFees`.

The renewal verifiers were not changed, and it is worth knowing why that is safe. `EthDomainRenewalVerificationService` hashes the on chain input with `this.web3.utils.keccak256(tx.input)` at `src/components/eth-domain-renewal/services/eth-domain-renewal-verification.service.ts` line 38 (and the SPACE ID verifier does the same at `bnb-arb-domain-renewal-verification.service.ts` line 41), using web3.js, while the prepare step hashes with `ethers.keccak256(data)` (ethers v6), and the confirm only succeeds if the two strings are exactly equal. Both libraries return lowercase `0x` hex for the same bytes, so the comparison still holds after the upgrade, but it is a cross library contract with no test of its own, and the day either library changed its output casing every renewal confirm would start failing with "Transaction calldata does not match the prepared renewal". Comparing with `.toLowerCase()` on both sides, as `deployment-verification.service.ts` line 104 already does, would make that impossible. The codebase wide migration story is in [../12-marketplace-v2/12-ethers-v6-migration-and-the-new-test-suite.md](../12-marketplace-v2/12-ethers-v6-migration-and-the-new-test-suite.md).
