# 05. Blockchain Infrastructure

## What this cluster covers

Every other cluster of notes in this project deals with a feature a normal SaaS backend would also have, users, orders, payments, an admin panel. This cluster is different, it is the plumbing underneath the one thing that makes Endless Domains a Web3 company rather than a normal domain registrar, the code that talks to actual blockchains. Nine folders make up this slice, `src/components/web3`, `src/components/alchmey`, `src/components/moralis`, `src/components/nft-collection`, `src/components/contract-deployment`, `src/components/blockchain-deployment`, and `src/components/listener`, plus the two Solidity contracts at the project root in `contracts/`, and a handful of one time build scripts in `src/scripts/`.

If you have never touched Web3 before, read the notes in the order below. The first file is a real primer, it does not assume you know what a smart contract, a transaction, or an ABI is, and it teaches those ideas using this codebase's own contracts rather than a toy example. Everything after that builds on it.

## Reading order

[01-blockchain-fundamentals-primer.md](01-blockchain-fundamentals-primer.md) starts from zero, what a blockchain actually is for this company's purposes, what a smart contract is, what compiling one produces, and what it means to deploy one, all explained through `contracts/EndlessToken.sol` and `contracts/EndlessCollection.sol`.

[02-web3js-and-ethersjs-two-libraries-one-job.md](02-web3js-and-ethersjs-two-libraries-one-job.md) explains the two JavaScript libraries this codebase uses to talk to a blockchain, `web3.js` and `ethers.js`, why both are present in the same `package.json`, and which one this codebase reaches for depending on whether it is reading existing on chain data or building a new transaction.

[03-solidity-contracts-and-the-compile-pipeline.md](03-solidity-contracts-and-the-compile-pipeline.md) walks through the two contract source files and the five build scripts in `src/scripts/`, showing exactly how a `.sol` file becomes the bytecode and ABI constants that the running server actually uses, and why that compilation happens once, by hand, long before any user request.

[04-contract-deployment-and-nft-collection-prepare-confirm-pattern.md](04-contract-deployment-and-nft-collection-prepare-confirm-pattern.md) covers `contract-deployment` and `nft-collection` together, because they are the same pattern applied to two different contract templates, a prepare step that builds an unsigned deployment transaction for the user's own wallet to sign, and a confirm step that checks what actually happened on chain.

[05-deployment-verification-the-shared-trust-boundary.md](05-deployment-verification-the-shared-trust-boundary.md) is a close read of `blockchain-deployment`'s single service, the seven step check that decides whether a submitted transaction hash is allowed to mark a deployment confirmed.

[06-alchemy-and-the-alchmey-folder.md](06-alchemy-and-the-alchmey-folder.md) explains what Alchemy is as a product, and how the `alchmey` folder's per chain services use its SDK to answer "what NFTs does this wallet own" without this backend ever running its own blockchain node.

[07-moralis-and-cross-chain-nft-lookups.md](07-moralis-and-cross-chain-nft-lookups.md) does the same for Moralis, the second, older data provider this codebase leans on, and shows where the two providers overlap and where each one is still the only path for a particular chain.

[08-per-chain-web3-services-and-ipfs.md](08-per-chain-web3-services-and-ipfs.md) covers the rest of the `web3` folder, the small per chain services (`ens-web3`, `arb-web3`, `ud-web3`, `bnb-web3`) that read directly from a contract using `web3.js`, the `web3-transaction` module that is really just a database log table, and the `ipfs` module that has nothing to do with blockchains reading or writing at all, it pins a customer's static website to IPFS through Pinata.

[09-listener-what-it-really-is.md](09-listener-what-it-really-is.md) closes the cluster with a finding worth sitting with, the folder is named `listener`, and the obvious guess is that it subscribes to live blockchain events like a token transfer. It does not. It is a small, generic, database backed registry for starting and stopping named cron jobs at runtime. This file shows the evidence and explains why that distinction matters.

## A habit worth carrying from note to note

Almost everything in this cluster follows one shape, a request comes in, the server either reads something from a blockchain through a paid third party API (Alchemy, Moralis) or a direct RPC connection (`web3.js`, `ethers.js`), or it builds a transaction for a user's own wallet to sign, never holding a private key that could move a user's funds on its own. Keep that shape in mind, once you recognize it in one file, you will recognize it in nearly every other file in this cluster.
