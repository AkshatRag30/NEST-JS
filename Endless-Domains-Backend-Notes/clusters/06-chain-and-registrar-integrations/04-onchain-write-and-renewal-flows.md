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
const data = this.encodeRenewCalldata(quote.label, dto.durationSeconds);
const preparedCalldataHash = ethers.utils.keccak256(data);
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
