# Smart Contract Build Notes

## Step 1: Install npm and Node

You need npm and node. Confirm they are installed with:

```bash
$ node -v
$ npm -v
```

## Step 2: Create a Hardhat Project

You'll be working with Hardhat. Install it with:

```bash
$ mkdir usdc-balance && cd usdc-balance
$ npm init -y
$ npm install –save-dev hardhat
$ npx hardhat
```

Once the Hardhat project is created, make sure everything is running:

```bash
$ npx hardhat test
```

## Step 3: Install MetaMask and Acquire Goerli ETH

You'll need MetaMask to interact with the Ethereum blockchain. Install it and create an account. Then, acquire some Goerli ETH from a faucet.

https://goerlifaucet.com/

## Step 4: Write the USDC Balance Contract

Create a new file called `contracts/USDCBalance.sol` and add the following code:

```solidity
//SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

import "hardhat/console.sol";

interface MainUsdcContractInterface {
    // Define signature of balanceOf
    function balanceOf(address owner) external view returns (uint);
}

// Contract that interacts with main USDC contract to get balance of a wallet
contract UsdcBalance {

    // Address of USDC contract
    address public usdcContractAddress;
    MainUsdcContractInterface public usdcContract;

    // Create a pointer to the USDC contract
    constructor(address _usdcContractAddress) {
        usdcContract = MainUsdcContractInterface(_usdcContractAddress);
    }

    // Function to get USDC balance
    function getUsdcBalance() public view returns (uint) {

        uint balance = usdcContract.balanceOf(msg.sender);
        return balance;
    }
}
```

Compile the contracts using:

```bash
$ npx hardhat compile
```

## Step 5: Configure Hardhat Project

We'll want to configure the Hardhat project to use Metamas to sign transactions.

In the root folder of your project, you will find a file called hardhat.config.js. Replace its contents with the following:

```javascript
require("@nomicfoundation/hardhat-toolbox");

/** @type import('hardhat/config').HardhatUserConfig */
module.exports = {
  solidity: "0.8.4",
  networks: {
    goerli: {
      url: "< GOERLI RPC URL >",
      accounts: ["< WALLET PRIVATE KEY >"],
    },
  },
};
```

You will need an RPC endpoint to send requests to Goerli. You can create this for free using a service like Infura or Alchemy.

## Step 6: Deploy and Test Contract

In the scripts folder, create a new file called run.js and add the following code:

```javascript
const hre = require("hardhat");

async function main() {
  // Get wallet address
  const [owner] = await hre.ethers.getSigners();

  // Define the official USDC contract address
  usdcContractAddress = "0x07865c6E87B9F70255377e024ace6630C1Eaa37F";

  const contract = await hre.ethers.deployContract("UsdcBalance", [
    usdcContractAddress,
  ]);

  await contract.waitForDeployment();
  console.log("USDC Balance contract deployed to: ", contract.target, "\n");

  // Get wallet's USDC balance
  let usdcBalance = await contract.connect(owner).getUsdcBalance();
  console.log(`USDC Balance of ${owner.address}: `, usdcBalance);
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.log(error);
    process.exit(1);
  });
```

Run the code using:

```bash
$ npx hardhat run scripts/run.js -–network goerli
```

If all goes well, you should see output that looks like this:

```bash
USDC Balance contract deployed to:  0x2B05Cb8fDaEDd01e777828fF6ac916A3E657828A
USDC Balance of 0x426b156dD2932A17629AA24A751C1A8C3fcdaC19: 51468456888
```
