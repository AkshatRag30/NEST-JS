# 02. web3.js and ethers.js, Two Libraries, One Job

## Why both are in the same `package.json`

Looking at the dependency list, this backend carries `web3` and `web3-utils` at version `^1.10.0`, and `ethers` at version `^6.13.4` (it was `^5.7.2` until commit `71fd2fec`, see the update at the end of this note), and the code also imports `@ethersproject/bignumber` and `@ethersproject/bytes` (smaller companion packages from the ethers v5 family). Be careful with that last pair, they are not actually listed in `package.json` at all, in the v5 era they arrived as dependencies of `ethers` itself, and today they only arrive because `alchemy-sdk` depends on them (`package-lock.json` lines 8713 to 8722), which the update section below explains. Both `web3.js` and `ethers.js` exist to solve the exact same underlying problem, letting a JavaScript program talk to an EVM compatible blockchain, whether that is reading data off it or sending a signed transaction to it. They are, in a real sense, competitors, most projects pick one and use it everywhere. This codebase did not, and reading through it, a clear pattern emerges for why.

## The pattern this codebase actually follows

Every file that only reads existing state from a chain, checking whether a domain name is available, looking up a token's owner or expiry, uses `web3.js`. Every file that needs to construct a transaction, encode constructor arguments, estimate gas, or verify a transaction's calldata and receipt after the fact, uses `ethers.js`. This is not stated anywhere as a rule, but it holds consistently across every file in this cluster.

Take `ens-web3.service.ts`, a pure read path:

```ts
import Web3 from 'web3';
import { AbiItem } from 'web3-utils';
...
async connectWeb3(): Promise<any> {
    return new Web3(new Web3.providers.WebsocketProvider(this.INFURA_URL));
}

async checkDomainAvailability(domainName: string): Promise<boolean> {
    const ensContract = await this.getENSContract();
    const domainAvailability = await ensContract.methods.available(domainName).call();
    return domainAvailability;
}
```

`new Web3(...)` opens a connection to a node, here over a websocket, to an Infura endpoint (Infura is another node hosting provider, similar in spirit to Alchemy, covered in file 06). `new web3.eth.Contract(abi, address)` builds an object that knows how to talk to one specific deployed contract, using its ABI to figure out what functions exist. `.methods.available(domainName).call()` then does a "call," meaning it asks a node to run that function's logic and hand back the result, without ever needing a signature, because reading data costs nothing and changes nothing on chain. `arb-web3.service.ts` and `fetch-expiry.service.ts` follow the identical shape, just against a plain HTTP provider instead of a websocket one, and against different ABIs, `ARBRegistrarControllerContract.json`, `ENSBaseRegistrarContract.json`, `BNBBaseRegistrarContract.json`.

Now look at `contract-deployment.service.ts`, which needs to actually build something to be signed, not just read a value:

```ts
// src/components/contract-deployment/services/contract-deployment.service.ts (lines 61 to 68, ethers v6 form as of uat)
import { ethers } from 'ethers';
...
const supplyWei = ethers.parseUnits(dto.supply, OZ_ERC20_DECIMALS);

const iface = new ethers.Interface(OZ_ERC20_CONSTRUCTOR_ABI);
const encodedArgs = iface.encodeDeploy([dto.name, dto.symbol, supplyWei, ownerAddress]);
const data = OZ_ERC20_BYTECODE + encodedArgs.slice(2);

const gasEstimate = await this.estimateGas(chainConfig, data, signerAddress);
const preparedCalldataHash = ethers.keccak256(data);
```

(Until commit `71fd2fec` these lines read `ethers.utils.parseUnits`, `new ethers.utils.Interface` and `ethers.utils.keccak256`, the ethers v5 spellings; the update section at the bottom of this note explains the move.) `ethers.Interface` is built purely from the constructor's ABI fragment, and `encodeDeploy` turns a plain JavaScript array of arguments, a name, a symbol, a supply, an owner address, into the exact ABI encoded byte sequence a contract constructor expects to receive. Concatenating that onto the front of the compiled bytecode (after stripping ethers' own leading `0x` from the encoded args with `.slice(2)`, since the bytecode already supplies one) produces the complete `data` field of an unsigned contract creation transaction, precisely the mechanic described in file 01. `ethers.keccak256` then hashes that exact payload, a fingerprint this company stores and later checks against, covered fully in file 05.

`ethers.JsonRpcProvider` (spelled `ethers.providers.JsonRpcProvider` in the v5 era) shows up the same way in `estimateGas` and throughout `deployment-verification.service.ts`, wrapping a plain RPC URL fetched from AWS Secrets Manager, then calling methods like `provider.estimateGas(...)`, `provider.getTransaction(txHash)`, `provider.getTransactionReceipt(txHash)`, and `provider.getBlock(blockNumber)`, none of which need a private key, they are still reads, just reads phrased in ethers' API rather than web3.js's, because the rest of that file's logic (encoding, hashing) is already written against ethers.

## Why the split makes sense rather than being an accident

`web3.js` is the older of the two libraries, and its `contract.methods.someFunction(...).call()` style reads naturally once a contract object exists, which is exactly what the smaller, older per chain services in the `web3` folder needed, and probably why they were written against it first. `ethers.js` has a reputation for a cleaner, more explicit API around exactly the operations the newer `contract-deployment`, `nft-collection`, and `blockchain-deployment` modules need, ABI encoding utilities like `Interface.encodeDeploy`, `keccak256`, and typed provider and transaction objects. Reading the file dates implied by the surrounding comments (the deployment modules describe themselves as a later "phase" or "sprint" built on top of infrastructure from an earlier one), the honest read is that this is a codebase that reached for whichever library fit the job in front of it at the time, and never went back to unify the two, which is a completely normal thing to find in a multi year production system, not a sign of carelessness.

## The one thing neither library does here

Notice that in every single example above, across both libraries, this backend is never the party holding a private key and sending a transaction on a user's behalf for a deployment. `estimateGas` only simulates what a transaction would cost, it does not send anything. The actual signing and broadcasting of a deployment transaction happens in the user's own wallet, outside this backend entirely, this backend only ever prepares the unsigned `data` and later verifies what came back. That is not a limitation of web3.js or ethers.js, it is a deliberate architectural choice this company made, and it is the subject of the next two files.

## Where to go next

[03-solidity-contracts-and-the-compile-pipeline.md](03-solidity-contracts-and-the-compile-pipeline.md) shows exactly how the `OZ_ERC20_BYTECODE` and `OZ_ERC721_BYTECODE` constants referenced above actually come into existence, the one time, by hand, build step that turns the Solidity source into those strings.

## Update from the October 2026 uat pull

The biggest change to this note's subject is that `ethers` itself moved a whole major version. Commit `71fd2fec` ("Ether versoin 6", 11 September 2026) changed `package.json` from `"ethers": "^5.7.2"` to `"ethers": "^6.13.4"`, and `package-lock.json` now resolves it to `6.17.0`. Commit `26b1f0e4` ("Upgraded the ethers version", 14 September 2026) then finished the job in the renewal and ENS modules. `web3` stayed at `^1.10.0`, so the "web3.js reads, ethers writes" split described above still holds, it just means the ethers half of the codebase now speaks a different dialect. The full, codebase wide story, including a translation table and the new tests that pin the behaviour, lives in [../12-marketplace-v2/12-ethers-v6-migration-and-the-new-test-suite.md](../12-marketplace-v2/12-ethers-v6-migration-and-the-new-test-suite.md). Here is what matters for the files this note quotes.

The single most visible change is that ethers v6 deleted the `ethers.utils` and `ethers.providers` namespaces. Everything that used to hang off them is now a top level export. So `ethers.utils.Interface` became `ethers.Interface`, `ethers.utils.keccak256` became `ethers.keccak256`, `ethers.utils.parseUnits` became `ethers.parseUnits`, `ethers.utils.getAddress` became `ethers.getAddress`, `ethers.utils.id` became `ethers.id`, `ethers.utils.namehash` became `ethers.namehash`, and `ethers.providers.JsonRpcProvider` became `ethers.JsonRpcProvider`. The type names moved the same way, `ethers.providers.TransactionResponse` is now `ethers.TransactionResponse`, `ethers.providers.TransactionReceipt` is now `ethers.TransactionReceipt`, `ethers.providers.Block` is now `ethers.Block`, and `ethers.utils.LogDescription` is now `ethers.LogDescription`. The algorithms behind these functions did not change at all, keccak256 is still keccak256 and ABI encoding is still ABI encoding, which is why the new golden hash tests in `contract-deployment.service.spec.ts` line 109 and `nft-collection.service.spec.ts` line 119 can pin exact outputs.

The second change is about numbers. In v5 every big on chain integer came back as a `BigNumber` object with methods like `.toNumber()`, `.isZero()` and `.eq()`. In v6 they come back as native JavaScript `bigint` values, the same type you get from writing `42n`. That is why `estimateGas` in both deployment services changed like this:

```ts
// src/components/contract-deployment/services/contract-deployment.service.ts (lines 278 to 280)
const provider = new ethers.JsonRpcProvider(rpcUrl);
const estimate = await provider.estimateGas({ data, from: fromAddress });
return Number(estimate);
```

In v5 the last line was `return estimate.toNumber();`. A `bigint` has no `.toNumber()` method, so leaving the old line in place would have thrown `TypeError: estimate.toNumber is not a function` at runtime, which the surrounding `catch` would have silently converted into `DEFAULT_GAS_ESTIMATE_FALLBACK` on every single request. That is a nasty kind of failure because nothing visibly breaks, every user just gets the fallback gas number forever. The same file shape exists at `src/components/nft-collection/services/nft-collection.service.ts` lines 313 to 315. `Number()` is safe here because a gas estimate is always far below `Number.MAX_SAFE_INTEGER`.

A frontend developer should also know the one place where the old v5 library is still alive on purpose. `src/components/alchmey/arbAlchmey/arb-alchmey.servers.ts` line 11 still imports `BigNumber` from `@ethersproject/bignumber`, and `src/components/alchmey/fetch-expiry/fetch-expiry.service.ts` line 11 and `src/components/ud-integration/ud-integration.service.ts` line 27 still import `arrayify` from `@ethersproject/bytes`. These are the standalone v5 packages, not the `ethers` package, so the major version bump did not touch them, and the new specs `arb-alchmey.servers.spec.ts` and `keccak256-cross-agreement.spec.ts` exist precisely to prove that. The honest risk is that neither package is declared in `package.json`. Before the upgrade they were pulled in by `ethers@5` itself, and now they are only present because `alchemy-sdk` happens to depend on `@ethersproject/*` at `^5.7.0`. If `alchemy-sdk` is ever upgraded to a release that drops those dependencies, or npm stops hoisting them to the top of `node_modules`, the imports at those three lines will fail at boot with "Cannot find module". The fix is a one line addition of both packages to `dependencies`.

The last behavioural difference worth carrying in your head is how a v6 `JsonRpcProvider` treats an unreachable node. When it cannot detect the network on first use, v6 logs "JsonRpcProvider failed to detect network and cannot start up; retry in 1s" and keeps retrying in the background every second, rather than failing the call quickly the way v5 tended to. The newer marketplace v2 code knows this and passes `{ staticNetwork: true }` with an explicit chain id, with the comment at `src/components/marketplacev2/order/order.service.ts` lines 154 to 159 saying exactly why. None of the older services this cluster covers do that. `deployment-verification.service.ts` line 71, `contract-deployment.service.ts` line 278, `nft-collection.service.ts` line 313, `gm-verification.service.ts` line 178 and the four providers built in `recent-domain.service.ts` lines 53 to 57 all call `new ethers.JsonRpcProvider(url)` with no network hint. The concrete failure scenario is a misconfigured or dead RPC URL in Secrets Manager: instead of a quick "Failed to fetch transaction from chain" response, the request can hang until the HTTP layer gives up, and the log fills with the retry message once a second for every provider object created. File 05 of this cluster walks through what that means for the deployment verification fallback loop specifically.
