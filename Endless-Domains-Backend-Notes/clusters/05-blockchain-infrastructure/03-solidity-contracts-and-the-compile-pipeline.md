# 03. The Solidity Contracts and the Compile Pipeline

## Five scripts, run by hand, never at request time

`src/scripts/` holds five files relevant to this cluster, `compile-erc20-template.js`, `write-erc20-template-constant.js`, `compile-nft-template.js`, `write-nft-template-constant.js`, and `verify-nft-template.js`, plus the two JSON artifacts they produce as a side effect, `erc20-template-artifact.json` and `nft-template-artifact.json`. Every one of them opens with a comment making the same point in different words, this is a one time, manual step, never something that runs while the server is handling a real request. The comment on `compile-erc20-template.js` is explicit about when a person is expected to run it, "whenever `contracts/EndlessToken.sol` changes." Understanding this pipeline matters because it explains where the `OZ_ERC20_BYTECODE` and `OZ_ERC721_BYTECODE` constants used throughout `contract-deployment` and `nft-collection` actually come from, and why no `solc` compiler dependency is ever touched while the app is live and serving traffic.

## Step one, compiling the source

`compile-erc20-template.js` and `compile-nft-template.js` are nearly identical, each reads its matching `.sol` file straight off disk, feeds it to the `solc` npm package (a JavaScript build of the real Solidity compiler), and asks for two specific outputs, the ABI and the bytecode:

```js
const input = {
    language: 'Solidity',
    sources: {
        'EndlessToken.sol': { content: CONTRACT_SOURCE }
    },
    settings: {
        evmVersion: 'paris',
        optimizer: { enabled: true, runs: 200 },
        outputSelection: {
            '*': { '*': ['abi', 'evm.bytecode.object'] }
        }
    }
};
```

Two of these settings are worth pausing on because they show real engineering judgment rather than defaults left untouched. `evmVersion: 'paris'` deliberately targets an older version of the Ethereum Virtual Machine's instruction set, one that predates a newer instruction called `PUSH0`, and the comment explains exactly why, "kept compatible with the least modern chain in `nft-chain.config.ts`." Remember from `nft-chain.config.ts` (covered in file 04) that this company supports around eighteen different EVM chains, some of them small or newer layer two networks, and not all of them are guaranteed to support every newer EVM feature. Rather than compile once per chain, they compile once, deliberately targeting the lowest common denominator, so the exact same bytecode works everywhere. `optimizer: { enabled: true, runs: 200 }` tells `solc` to spend effort shrinking and speeding up the compiled bytecode, `runs: 200` is a tuning knob that assumes the contract's functions will be called roughly that many times over its lifetime, a reasonable default for a contract most of this company's users will interact with a handful of times, not thousands.

`findImports` is the other detail worth understanding, because Solidity's `import "@openzeppelin/contracts/token/ERC20/ERC20.sol";` line, seen back in file 01, only makes sense if something tells the compiler where that file physically lives on disk:

```js
function findImports(importPath) {
    try {
        const resolved = require.resolve(importPath, { paths: [path.join(__dirname, '../..')] });
        return { contents: fs.readFileSync(resolved, 'utf8') };
    } catch (err) {
        return { error: `File not found: ${importPath}` };
    }
}
```

This reuses Node's own module resolution (`require.resolve`) to find the installed `@openzeppelin/contracts` npm package inside `node_modules`, and hands its actual source code to `solc` on demand. This is exactly how compiling your own contract from audited building blocks, the choice explained in file 01, actually gets wired up mechanically, the imported OpenZeppelin source becomes part of what gets compiled and baked into the final bytecode.

Running either compile script prints the bytecode and the constructor only ABI straight to the terminal, and also writes a full JSON artifact, `{ abi, bytecode }`, to disk. The comment on `compile-erc20-template.js` is candid about that artifact's purpose, "dumped for a future `verify-erc20-template.js` to consume, not imported by application code, this file is a build artifact of the compile step."

## Step two, writing the constant a real service actually imports

`write-erc20-template-constant.js` and `write-nft-template-constant.js` take that JSON artifact and turn it into the actual TypeScript file the running server imports, `oz-erc20-template.constant.ts` or `oz-erc721-template.constant.ts`:

```js
const artifact = require('./erc20-template-artifact.json');
const bytecode = artifact.bytecode.startsWith('0x') ? artifact.bytecode : '0x' + artifact.bytecode;
const content = `... export const OZ_ERC20_BYTECODE = '${bytecode}'; ...`;
fs.writeFileSync(OUT_PATH, content);
```

The comment explaining why this second script exists at all, rather than just pasting the compiler's printed output by hand, is worth internalizing as a habit, "so the pasted-in bytecode is always exact, no manual copy/paste of a large hex string." A single mistyped character in a multi thousand character hex string would be an almost impossible bug to spot by eye, and would make every deployment using it fail in a confusing way. Automating the copy step removes an entire class of human error, a small, unglamorous script doing real, load bearing work.

## Step three, proving the compiled bytecode actually works

`verify-nft-template.js` is the most interesting of the five, because it does not touch a real blockchain at all, it spins up an entirely local, in memory fake one using the `ganache` package, and runs the exact same deployment path the real service will run against it:

```js
const provider = new ethers.providers.Web3Provider(ganache.provider({ logging: { quiet: true } }));
const signer = provider.getSigner(0);
...
const iface = new ethers.utils.Interface(artifact.abi.filter((f) => f.type === 'constructor'));
const encodedArgs = iface.encodeDeploy(['Test Collection', 'TEST', 'ipfs://fake-metadata-hash', ownerAddress]);
const data = artifact.bytecode + encodedArgs.slice(2);

const tx = await signer.sendTransaction({ data });
const receipt = await tx.wait();
```

The comment above the script names this precisely, "the EXACT same data-construction path as `NftCollectionService.prepareCollection`." This is a smoke test for the compile pipeline itself, does the bytecode this team is about to hard code into their production constants actually deploy successfully, does `safeMint` actually work, does `tokenURI` return what the contract's own doc comment promises. It goes as far as actually minting a token and reading back `ownerOf` and `tokenURI` to confirm the collection level metadata URI quirk from file 01 behaves as documented, then exits with a clear pass or fail message. Ganache making this possible without touching a real, gas costing chain, or waiting for real block times, is exactly the kind of tool worth remembering exists, a local, disposable EVM you can spin up and throw away entirely inside a test script.

## The shape of the whole pipeline

Put together, the flow for either contract is always the same four steps, edit the `.sol` file, run its `compile-*.js` script to get fresh bytecode and ABI, run its `write-*-template-constant.js` script to turn that into the TypeScript constant file the real service imports, and, for the NFT template specifically, run `verify-nft-template.js` once more to prove the new bytecode still deploys and behaves correctly before trusting it in production. None of this happens while a user is on the site, by the time `ContractDeploymentService.prepareContract` or `NftCollectionService.prepareCollection` run, `OZ_ERC20_BYTECODE` and `OZ_ERC721_BYTECODE` are already just plain string constants sitting in compiled code, exactly as fast and reliable as any other constant in the file.

## Where to go next

[04-contract-deployment-and-nft-collection-prepare-confirm-pattern.md](04-contract-deployment-and-nft-collection-prepare-confirm-pattern.md) picks up exactly where this leaves off, showing how those bytecode constants get combined with a specific user's chosen name, symbol, and supply into a real, unsigned deployment transaction.
