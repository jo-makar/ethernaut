# Ethernaut

## Setup

Install and run locally, since the version on https://ethernaut.openzeppelin.com is in read-only mode

```sh
# Install nvm

git clone https://github.com/OpenZeppelin/ethernaut.git
cd ethernaut

nvm install # Uses version specified in .nvmrc
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

# 00 Hello Ethernaut

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

