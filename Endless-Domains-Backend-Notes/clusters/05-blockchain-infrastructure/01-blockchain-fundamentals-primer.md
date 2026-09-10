# 01. Blockchain Fundamentals, Taught From This Codebase

## Why this file exists before any of the others

Every note after this one assumes you already know what a smart contract is, what compiling one produces, and what it means to deploy one. Rather than teach those ideas abstractly, this file teaches them using the two real Solidity files sitting in this project's own `contracts/` folder, `EndlessCollection.sol` and `EndlessToken.sol`. If you have never touched Web3 before, read this one slowly, everything else in this cluster leans on it.

## What a blockchain is, for the purposes of this backend

Set aside the wider debates about blockchains for a moment and think about what this specific company needs one for. A traditional domain registrar like GoDaddy keeps a database row that says "this domain belongs to this account." Endless Domains wants that ownership record to live somewhere no single company controls, a blockchain, which is really just a shared, append only ledger that thousands of independent computers around the world all keep an identical copy of, and all agree on the current state of through a shared set of rules. Ownership of a domain, in this world, is represented as ownership of a token, and owning a token is nothing more than your wallet address being the one recorded as its owner inside a piece of code running on that shared ledger. That piece of code is a smart contract, and it is the single most important idea in this entire cluster of notes.

## What a smart contract actually is

A smart contract is a program, written in a language called Solidity here, that lives permanently on a blockchain once it is deployed, and that anyone can call the functions of, subject to whatever rules the code itself enforces. Look at the shorter of this company's two contracts, `contracts/EndlessToken.sol`:

```solidity
contract EndlessToken is ERC20, Ownable {
    constructor(
        string memory name_,
        string memory symbol_,
        uint256 initialSupply_,
        address initialOwner_
    ) ERC20(name_, symbol_) Ownable(initialOwner_) {
        _mint(initialOwner_, initialSupply_);
    }
}
```

This is a fungible token contract, the same category of thing as any ERC20 token you may have heard of. `ERC20` and `Ownable` are not written here, they are imported from a widely used, independently audited library called OpenZeppelin, and this company's own contract is really just those two proven building blocks glued together with one constructor. The constructor runs exactly once, at the moment the contract is deployed, and this one takes a name, a symbol, a total supply, and an address that should own the whole thing, then calls `_mint`, which is the OpenZeppelin function that actually creates that supply and credits it to `initialOwner_`. Notice there is no separate `mint` function anywhere else in the file, the comment above the contract spells out why deliberately, "there is no separate mint function," the entire supply is created once, at construction, and never again. That is a real design decision with real consequences, whoever deploys this contract decides the total supply forever, up front.

The second contract, `contracts/EndlessCollection.sol`, is a non fungible token, an NFT collection, contract, meaning each token it can create is unique rather than interchangeable:

```solidity
contract EndlessCollection is ERC721, Ownable {
    string private _collectionURI;

    constructor(
        string memory name_,
        string memory symbol_,
        string memory baseTokenURI_,
        address initialOwner_
    ) ERC721(name_, symbol_) Ownable(initialOwner_) {
        _collectionURI = baseTokenURI_;
    }

    function safeMint(address to, uint256 tokenId) external onlyOwner {
        _safeMint(to, tokenId);
    }

    function tokenURI(uint256 tokenId) public view override returns (string memory) {
        _requireOwned(tokenId);
        return _collectionURI;
    }
}
```

Here the constructor does not mint anything, it just records a name, a symbol, and a single metadata URI, `_collectionURI`, that every token in the collection will share. Minting an actual token happens later, through `safeMint`, and `onlyOwner` is an OpenZeppelin guard that makes that function revert if anyone other than the contract's owner address tries to call it. The comment above the contract explains a real quirk worth remembering, this collection's `tokenURI` function always returns the one collection level URI regardless of which token you ask about, which is not how most NFT collections behave, most give each token its own metadata address. That is a deliberate, documented choice by this company's engineers, not a bug, and it is exactly the kind of detail you would only catch by reading the contract itself rather than assuming NFTs "just work a certain way."

## What compiling a contract actually produces

Solidity source code cannot run on a blockchain directly, any more than a `.ts` file can run directly in a browser without being compiled to JavaScript first. A Solidity compiler, `solc` here (see `package.json`, and every compile script in `src/scripts/`), takes the `.sol` source and produces two very different, and both essential, outputs.

The first is bytecode, a long string of raw machine level instructions for the Ethereum Virtual Machine, the shared computer that every node on an EVM compatible chain runs to execute contract code identically. This is what actually gets sent to the network when a contract is deployed, and what actually gets stored and run on chain forever after. You will see it referenced throughout this cluster as `OZ_ERC20_BYTECODE` and `OZ_ERC721_BYTECODE`, a very long hexadecimal string starting with `0x`, deliberately not reproduced in full in these notes since it is compiled output baked directly into this company's own constant files, but conceptually it is no different from a compiled `.exe`, unreadable to a person, completely readable to the machine meant to run it.

The second output is the ABI, the Application Binary Interface, and this one you actually will read directly, because it is a plain JSON description of every function and constructor the contract exposes, its name, its parameters, their types. Compare `compile-erc20-template.js`'s comment about what it extracts:

```js
const contract = output.contracts['EndlessToken.sol'].EndlessToken;
const bytecode = '0x' + contract.evm.bytecode.object;
const abi = contract.abi;
const constructorAbi = abi.filter((f) => f.type === 'constructor');
```

The bytecode is the "what to run," the ABI is the "how to call it," a translation layer that lets a JavaScript backend that has never seen the Solidity source describe, correctly, what arguments a function expects and in what order, purely by reading this JSON. Every time this codebase later builds a call to a contract function, `available(domainName)` on an ENS registrar, `safeMint(to, tokenId)` on a deployed collection, it is the ABI, not the bytecode, doing the work of making that call type safe and correctly encoded.

## Why this company compiles its own token and NFT templates

It would be far simpler for Endless Domains to just point at some other company's already deployed ERC20 or ERC721 contract and reuse it. They do not, and the doc comments left directly in the Solidity files explain why, in both cases the constructor signature is deliberately fixed and cross referenced against a specific TypeScript constant file:

> Constructor signature is fixed by `contract-deployment/constants/oz-erc20-template.constant.ts` (OZ_ERC20_CONSTRUCTOR_ABI) and `ContractDeploymentService.prepareContract`'s encodeDeploy call, changing it here requires updating both.

Owning the exact source, compiled by this team, with a pinned OpenZeppelin version and a pinned compiler version, means the backend can hard code the exact bytecode and constructor ABI it expects, and trust that a `name`, `symbol`, `supply`, `owner` tuple will always produce the same shape of contract, every single time, on every chain this company supports. If they instead deployed via some third party factory contract, they would be trusting that factory's own code and its own constructor shape, and would have no control if that factory ever changed. Compiling their own tiny, audited template gives Endless Domains a stable, predictable building block their own deployment flow (covered in file 04) can rely on completely.

## What "deploying a contract" actually means, as an operation

This is worth being precise about, because it is easy to imagine deployment as some special blockchain only operation, and it is not, it is a normal transaction with one particular property. A transaction on an EVM chain, at the lowest level, either sends value to an existing address, or it sends `data` to an existing address, or, the case that matters here, it sends `data` with no destination address at all. When a transaction's `to` field is empty and it carries `data`, every node on the network treats that as an instruction to create a new contract at a fresh address, derived deterministically from the sender's address and how many transactions that sender has previously sent, and it runs the `data` as that new contract's bytecode plus its ABI encoded constructor arguments. The receipt that comes back once the transaction is mined includes a `contractAddress` field, that fresh address, now permanently associated with that piece of code.

You can see this exact mechanic acknowledged directly in this codebase's own verification logic, in `deployment-verification.service.ts`:

```ts
// Step 5: Contract deployment check
if (!receipt.contractAddress) {
    return {
        valid: false,
        reason: 'Transaction did not deploy a contract. Please submit a valid deployment transaction.'
    };
}
```

If `receipt.contractAddress` is empty, whatever transaction hash the user submitted was not a contract creation transaction at all, it was an ordinary transfer or contract call, and this backend refuses to treat it as a deployment. File 04 shows exactly how this company builds that `data` payload, bytecode plus encoded constructor arguments, and hands it to the user's own wallet to sign, and file 05 shows the full checklist this verification service runs before trusting that a signed transaction really did what it claims.

## Where to go next

[02-web3js-and-ethersjs-two-libraries-one-job.md](02-web3js-and-ethersjs-two-libraries-one-job.md) introduces the two JavaScript libraries this codebase actually uses to talk to a blockchain, and shows, with real examples from this codebase, when it reaches for one over the other.
