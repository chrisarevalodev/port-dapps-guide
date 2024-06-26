# Port an Ethereum dApp to Rootstock Guide
## Install [hardhat](https://hardhat.org/) globally

```bash
npm i -g hardhat
```

## Initialize hardhat
### Create a folder for your project and initialize a hardhat project

```bash
mkdir rsk-hardhat-example
cd rsk-hardhat-example
npx hardhat init
```
### Select *Create a TypeScript project* on the hardhat CLI
```bash
888    888                      888 888               888
888    888                      888 888               888
888    888                      888 888               888
8888888888  8888b.  888d888 .d88888 88888b.   8888b.  888888
888    888     "88b 888P"  d88" 888 888 "88b     "88b 888
888    888 .d888888 888    888  888 888  888 .d888888 888
888    888 888  888 888    Y88b 888 888  888 888  888 Y88b.
888    888 "Y888888 888     "Y88888 888  888 "Y888888  "Y888

👷 Welcome to Hardhat v2.22.5 👷‍

? What do you want to do? …
  Create a JavaScript project
❯ Create a TypeScript project
  Create a TypeScript project (with Viem)
  Create an empty hardhat.config.js
  Quit
```
### Select the project root (press enter)
```bash
✔ What do you want to do? · Create a TypeScript project
? Hardhat project root: › /path/to/your/project/rsk-hardhat-example
```

### Add a .gitignore in case you need it
```bash
? Hardhat project root: › /path/to/your/project/rsk-hardhat-example
? Do you want to add a .gitignore? (Y/n) › y
```

### Select that you'd like to install dependencies with `npm`
```bash
✔ Do you want to add a .gitignore? (Y/n) · y
? Do you want to install this sample project's dependencies with npm (hardhat @nomicfoundation/hardhat-toolbox)? (Y/n) › y
```

### Open your code editor
For this example we'll be using [Visual Studio Code](https://code.visualstudio.com/) but feel free to use the one you prefer
```bash
code .
```

## Configure environment and RSK networks (mainnet and testnet)
By now your hardhat project should have 4 main artifacts besides the basic [Node](https://nodejs.org/en/) configuration:
```bash
contracts/
ignition/modules/
test/
hardhat.config.js
```
> [!NOTE]
> The version of Hardhat we'll be using in this example is v2.22.5. For this version, the default tool for managing deployments is [Hardhat Ignition](https://hardhat.org/ignition/docs/getting-started).

### Install Hardhat Ignition and typescript
```bash
npm install --save-dev @nomicfoundation/hardhat-ignition-ethers typescript
```
### And import Hardhat Ignition on the `hardhat.config.ts`
```ts
import "@nomicfoundation/hardhat-ignition-ethers";
```
### Configure the RSK networks
By now your `hardhat.config.ts` should look something like this

```ts
import { HardhatUserConfig } from "hardhat/config";
import "@nomicfoundation/hardhat-toolbox";
import "@nomicfoundation/hardhat-ignition-ethers";

const config: HardhatUserConfig = {
  solidity: "0.8.24",
};

export default config;
```
In order to configure the RSK networks we'll need: an RPC url for both mainnet and testnet and a Private Key of the account that will deploy the contracts. To get the the RPCs go to the [RPC API dashboard](https://dashboard.rpc.rootstock.io/dashboard) from Rootstock Labs, create an account if you don't have one and get an API key for RSK testnet and another for RSK mainnet. 

The mainnet RPC url should look similar to this:
```
https://rpc.mainnet.rootstock.io/<API-KEY>
```
The testnet RPC url like this
```
https://rpc.testnet.rootstock.io/<API-KEY>
```
And if you don't know how to get the Private Key of your wallet, here's a [tutorial](https://support.metamask.io/managing-my-wallet/secret-recovery-phrase-and-private-keys/how-to-export-an-accounts-private-key/) on how to do it on [Metamask](https://metamask.io/). Also, if you haven't added RSK mainnet or testnet to your Metamask Wallet, you can do it by clicking the **Add Rootstock** or **Add Rootstock Testnet** buttons in the footer of [mainnet explorer](https://rootstock.blockscout.com/) or [testnet explorer](https://rootstock-testnet.blockscout.com/).

### Store the RPC urls and the PK
For storing the RPC urls securely you can use a .env file or the [hardhat configuration variables](https://hardhat.org/hardhat-runner/docs/guides/configuration-variables). For this example we'll be using the second one. To store the type this in the terminal of the project's root:
```
npx hardhat vars set MAINNET_RPC_URL
```
And enter the value after pressing enter. Repeat this step with the other two:
```
npx hardhat vars set TESTNET_RPC_URL
✔ Enter value: ********************************
npx hardhat vars set PRIVATE_KEY
✔ Enter value: *************************************
```
### Set the configuration
Now use all the things we've used and stored so your `hardhat.config.ts` looks like this:
```ts
import { HardhatUserConfig, vars } from "hardhat/config";
import "@nomicfoundation/hardhat-toolbox";
import "@nomicfoundation/hardhat-ignition-ethers";

const MAINNET_RPC_URL = vars.get("MAINNET_RPC_URL");
const TESTNET_RPC_URL = vars.get("TESTNET_RPC_URL");
const PRIVATE_KEY = vars.get("PRIVATE_KEY");

const config: HardhatUserConfig = {
  solidity: "0.8.24",
  networks: {
    rskMainnet: {
      url: MAINNET_RPC_URL,
      chainId: 30,
      gasPrice: 60000000,
      accounts: [PRIVATE_KEY],
    },
    rskTestnet: {
      url: TESTNET_RPC_URL,
      chainId: 31,
      gasPrice: 60000000,
      accounts: [PRIVATE_KEY],
    },
  },
};

export default config;

```
And the configuration is completed, now let's bring the contracts from Ethereum.

## Copy Ethereum Contract Code and Tests
We'll copy the contracts from Ethereum and it's tests to our RSK hardhat project. The contract is the following and will be inside `contracts` folder so the route would be `contracts/SimpleStorage.sol`
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract SimpleStorage {
    uint256 public favoriteNumber;

    function store(uint256 _favoriteNumber) public  {
        favoriteNumber = _favoriteNumber;
    }
}
```
The same with the test. The file will be named `SimpleStorage.ts` inside the `test` folder so the result is `test/SimpleStorage.ts`
```ts
import { loadFixture } from "@nomicfoundation/hardhat-toolbox/network-helpers";
import { expect } from "chai";
import hre from "hardhat";

describe("SimpleStorage", function () {
  async function deploySimpleStorageFixture() {
    const [owner] = await hre.ethers.getSigners();

    const SimpleStorage = await hre.ethers.getContractFactory("SimpleStorage");
    const simpleStorage = await SimpleStorage.deploy();

    return { simpleStorage, owner };
  }

  describe("Deployment", function () {
    it("Should deploy and initialize favoriteNumber to 0", async function () {
      const { simpleStorage } = await loadFixture(deploySimpleStorageFixture);

      expect(await simpleStorage.favoriteNumber()).to.equal(0);
    });
  });

  describe("Store", function () {
    it("Should store the value 42 and retrieve it", async function () {
      const { simpleStorage } = await loadFixture(deploySimpleStorageFixture);

      const storeTx = await simpleStorage.store(42);
      await storeTx.wait();

      expect(await simpleStorage.favoriteNumber()).to.equal(42);
    });

    it("Should store a different value and retrieve it", async function () {
      const { simpleStorage } = await loadFixture(deploySimpleStorageFixture);

      const storeTx = await simpleStorage.store(123);
      await storeTx.wait();

      expect(await simpleStorage.favoriteNumber()).to.equal(123);
    });
  });
});

```
## Compile, Test and Deploy your contract
### Compile
In order to compile your contract, type this in your terminal:
```
npx hardhat compile
```
And you should get something like this:
```
Generating typings for: 1 artifacts in dir: typechain-types for target: ethers-v6
Successfully generated 6 typings!
Compiled 1 Solidity file successfully (evm target: paris).
```
### Test
For testing out your contract(s) write this command in the console:
```
npx hardhat test
```
And you should get something similar to this:
```
SimpleStorage
    Deployment
      ✔ Should deploy and initialize favoriteNumber to 0
    Store
      ✔ Should store the value 42 and retrieve it
      ✔ Should store a different value and retrieve it


  3 passing (286ms)
```
### Deploy
The requirement for deploying in mainnet or testnet is having enough balance in the native token. In this example we'll be deploying the contract on testnet.

> [!NOTE]
> To get testnet tokens for deploying your contract go to the [faucet](https://faucet.rootstock.io/).

#### Create the deployment script
Create a file named `SimpleStorage.ts` inside the folder `ignition/modules`. Then paste this in that file:

```ts
import { buildModule } from "@nomicfoundation/hardhat-ignition/modules";

const SimpleStorageModule = buildModule("SimpleStorageModule", (m) => {
  const simpleStorage = m.contract("SimpleStorage");

  return { simpleStorage };
});

export default SimpleStorageModule;
```

This is a script of typescript that uses hardhat ignition to deploy the `SimpleStorage` contract in a declarative way. After you paste that code and save the changes, type this in your terminal tto deploy the contract:

```
npx hardhat ignition deploy ignition/modules/SimpleStorage.ts --network rskTestnet
```
And you should get something like this:
```
✔ Confirm deploy to network rskTestnet (31)? … yes
Hardhat Ignition 🚀

Resuming existing deployment from ./ignition/deployments/chain-31

Deploying [ SimpleStorageModule ]

Batch #1
  Executed SimpleStorageModule#SimpleStorage

[ SimpleStorageModule ] successfully deployed 🚀

Deployed Addresses

SimpleStorageModule#SimpleStorage - 0x3570c42943697702bA582B1ae3093A15D8bc2115
```
> [!TIP]
> If you get an error like `IgnitionError: IGN401` try running the command again.

If you want to deploy your contract on mainnet change `rskTestnet` to `rskMainnet` in the last command we ran and make sure you have [RBTC](https://dev.rootstock.io/rsk/rbtc/) available on your wallet.

## Congrats! You deployed your contract on Rootstock!
Now go to [https://explorer.testnet.rootstock.io/](https://explorer.testnet.rootstock.io/) and paste your contract address in the search bar to verify the deployment was successful. If you deployed your contract on mainnet look in [https://explorer.rootstock.io/](https://explorer.rootstock.io/)
