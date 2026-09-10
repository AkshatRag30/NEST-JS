# 02. web3.js and ethers.js, Two Libraries, One Job

## Why both are in the same `package.json`

Looking at the dependency list, this backend carries `web3` and `web3-utils` at version `^1.10.0`, and `ethers` at version `^5.7.2`, alongside `@ethersproject/bignumber` and `@ethersproject/bytes` (smaller companion packages that ship as part of the ethers.js family). Both `web3.js` and `ethers.js` exist to solve the exact same underlying problem, letting a JavaScript program talk to an EVM compatible blockchain, whether that is reading data off it or sending a signed transaction to it. They are, in a real sense, competitors, most projects pick one and use it everywhere. This codebase did not, and reading through it, a clear pattern emerges for why.

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
import { ethers } from 'ethers';
...
const iface = new ethers.utils.Interface(OZ_ERC20_CONSTRUCTOR_ABI);
const encodedArgs = iface.encodeDeploy([dto.name, dto.symbol, supplyWei, ownerAddress]);
const data = OZ_ERC20_BYTECODE + encodedArgs.slice(2);

const gasEstimate = await this.estimateGas(chainConfig, data, signerAddress);
const preparedCalldataHash = ethers.utils.keccak256(data);
```

`ethers.utils.Interface` is built purely from the constructor's ABI fragment, and `encodeDeploy` turns a plain JavaScript array of arguments, a name, a symbol, a supply, an owner address, into the exact ABI encoded byte sequence a contract constructor expects to receive. Concatenating that onto the front of the compiled bytecode (after stripping ethers' own leading `0x` from the encoded args with `.slice(2)`, since the bytecode already supplies one) produces the complete `data` field of an unsigned contract creation transaction, precisely the mechanic described in file 01. `ethers.utils.keccak256` then hashes that exact payload, a fingerprint this company stores and later checks against, covered fully in file 05.

`ethers.providers.JsonRpcProvider` shows up the same way in `estimateGas` and throughout `deployment-verification.service.ts`, wrapping a plain RPC URL fetched from AWS Secrets Manager, then calling methods like `provider.estimateGas(...)`, `provider.getTransaction(txHash)`, `provider.getTransactionReceipt(txHash)`, and `provider.getBlock(blockNumber)`, none of which need a private key, they are still reads, just reads phrased in ethers' API rather than web3.js's, because the rest of that file's logic (encoding, hashing) is already written against ethers.

## Why the split makes sense rather than being an accident

`web3.js` is the older of the two libraries, and its `contract.methods.someFunction(...).call()` style reads naturally once a contract object exists, which is exactly what the smaller, older per chain services in the `web3` folder needed, and probably why they were written against it first. `ethers.js` has a reputation for a cleaner, more explicit API around exactly the operations the newer `contract-deployment`, `nft-collection`, and `blockchain-deployment` modules need, ABI encoding utilities like `Interface.encodeDeploy`, `keccak256`, and typed provider and transaction objects. Reading the file dates implied by the surrounding comments (the deployment modules describe themselves as a later "phase" or "sprint" built on top of infrastructure from an earlier one), the honest read is that this is a codebase that reached for whichever library fit the job in front of it at the time, and never went back to unify the two, which is a completely normal thing to find in a multi year production system, not a sign of carelessness.

## The one thing neither library does here

Notice that in every single example above, across both libraries, this backend is never the party holding a private key and sending a transaction on a user's behalf for a deployment. `estimateGas` only simulates what a transaction would cost, it does not send anything. The actual signing and broadcasting of a deployment transaction happens in the user's own wallet, outside this backend entirely, this backend only ever prepares the unsigned `data` and later verifies what came back. That is not a limitation of web3.js or ethers.js, it is a deliberate architectural choice this company made, and it is the subject of the next two files.

## Where to go next

[03-solidity-contracts-and-the-compile-pipeline.md](03-solidity-contracts-and-the-compile-pipeline.md) shows exactly how the `OZ_ERC20_BYTECODE` and `OZ_ERC721_BYTECODE` constants referenced above actually come into existence, the one time, by hand, build step that turns the Solidity source into those strings.
