This goes into package.json

```js
"deploy-lock-rinkeby": "yarn compile && yarn compile:mocks && buidler --config buidler.config.js --network rinkeby deploy-lock-minter-testnet"
```

This goes into `buidler.config.json`

```js
const { DeployLockMinterTestnet } = require('./deploy/deploy-lock-minter-testnet')


task('deploy-lock-minter-testnet', 'This deploys ANJLockMinter')
  .setAction(DeployLockMinterTestnet)
```

This goes into `deploy-lock-minter-testnet`

```js
const { bigExp } = require('@aragon/contract-helpers-test')

async function DeployLockMinterTestnet({ deploy }) {


}   



module.exports = {
    DeployLockMinterTestnet
}

// yarn deploy-protocol --network rinkeby
// yarn deploy-lock-rinkeby

```

----

### Rinkeby Deployment............


* Make sure that contracts are correct on the `Aragon Protocol` Repository.  
* `cd evm && yarn prepublishOnly && cd ../deployment && rm -rf scripts/1-deploy-protocol/output/protocol.rinkeby.json && yarn deploy-protocol --network rinkeby`.

The above compiles the contracts, and deploys them on the rinkeby. The deployed protocol uses the following `ANTv2` Address(`0xf0f8D83CdaB2F9514bEf0319F1b434267be36B5c`)
and `DAI` address(`0xb08E32D658700f768f5bADf0679E153ffFEC42e6`).

* Once the above is done, we copy the `GuardianRegistry` address and go to `Aragon-Network-Token`.

```js
const ANJAddress                = "0x96286BbCac30Cef8dCB99593d0e28Fabe95F3572";
const ANTv2Address              = "0xf0f8D83CdaB2F9514bEf0319F1b434267be36B5c";
const ANTv2MultiMinterAddress   = "0xF64bf861b8A85927FAdd9724E80C2987f82a9259";
const GuardianRegistryAddress   = "===============COPY HERE===================";

const ANJLockMinter  = artifacts.require('ANJLockMinter')

const ANJLockMinterInstance = await ANJLockMinter.new(
      ANTv2MultiMinterAddress, ANTv2Address, 
      ANJAddress, GuardianRegistryAddress
);

console.log(ANJLockMinterInstance.address, " ANJ Lock Minter");
```

This gives us the address of `ANJLockMinter`.

* Now, we go to the `Aragon Protocol` and run the following:

```js

const utils = this.environment.web3.utils;

let a = await this.protocol.grant(
  // the first argument - the address of GuardiansRegistry
  // the second argument - the signature of stackAndActivate
  utils.keccak256(utils.encodePacked("0x5599b28bf4c75F75B887C0FdD83d741ECA06C71d", "0x2b7f012e")),
  "=======COPY HERE THE ADDRESS ACQUIRED IN THE PREVIOUS STEP=================="
)

let b = await this.protocol.grant(
  // the first argument - the address of GuardiansRegistry
  // the second argument - the signature of stackAndActivate
  utils.keccak256(utils.encodePacked("0x5599b28bf4c75F75B887C0FdD83d741ECA06C71d", "0x51d2f186")),
  "=======COPY HERE THE ADDRESS ACQUIRED IN THE PREVIOUS STEP==================" // the address of ANJLockMinter
)
```

This will give the authorization to the `ANJLockMinter` so it can call the `stackAndActivate` and `lockActivation` on the `GuardiansRegistry`.

* Now, we go back to `Aragon Network Token` and run the following:


```js

const ANJLockMinter     = artifacts.require('ANJLockMinter')
const ANTv2MultiMinter  = artifacts.require('ANTv2MultiMinter');
const ANJ               = artifacts.require('ANJ');

const ANTv2MultiMinterAddress   = "0xF64bf861b8A85927FAdd9724E80C2987f82a9259";
const ANJAddress                = "0x96286BbCac30Cef8dCB99593d0e28Fabe95F3572";

const ANJLockMinterInstance  = await ANJLockMinter.at("=========== COPY ANJ LOCK MINTER ADDRESS ACQUIRED PREVIOUSLY ABOVE");
const MultiMinterInstance    = await ANTv2MultiMinter.at(ANTv2MultiMinterAddress);
const ANJInstance            = await ANJ.at(ANJAddress);

await MultiMinterInstance.addMinter(ANJLockMinterInstance.address);

await ANJInstance.approveAndCall(ANJLockMinterInstance.address, bigExp(9000, 18), "0x000000");


```

This approves 9000 ANJ tokens and calls `ANJLockMinter` which calls `stakeAndActivate` and `lockActivation` on the 9000 * 0.044 amount.....


*  At this point, Let's check the balance of the user(deployer - who called `approveAndCall`) on the `GuardiansRegistry` .

```js
let k = await this.registry.detailedBalanceOf("The address of whoever called ApproveAndCall");
console.log(k['active'].toString(), k['available'].toString(), k['locked'].toString(), k['pendingDeactivation'].toString());
```

In our case, it will be `396000000000000000000 0 0 0`.


* Now, we have to do something so that we can make sure that we can unlock the money.

Let's try to manage so that we can unlock and get back 120 ANT on our address.

```js
await this.registry.unlockActivation(
  "========GuardianAddress=======", // whoever wants to unlock the money.
  "========ANJLockMinterAddress=======", // ANJLockMinter
  bigExp(120, 18),
  true
);
```


This creates a `DeactivationRequest`. Let's check the balances again..

```js
let k = await this.registry.detailedBalanceOf("0x94C34FB5025e054B24398220CBDaBE901bd8eE5e");
console.log(k['active'].toString(), k['available'].toString(), k['locked'].toString(), k['pendingDeactivation'].toString());
```

This now shows `276000000000000000000 0 0 120000000000000000000`. So, everything looks good, but it turns out that 
`120000000000000000000` is `pendingDeactivation` balance, so, we will not be able to call `unstake` yet.

* Then, we call

```js
await this.registry.processDeactivationRequest(
   "0x94C34FB5025e054B24398220CBDaBE901bd8eE5e" // The address of the user|guardian.
)
```

After this, we can see that balances change like this:

```
276000000000000000000 120000000000000000000 0 0
```


* Final Step is this:

```js
await this.registry.unstake(
    "0x94C34FB5025e054B24398220CBDaBE901bd8eE5e", // guardian | user
    bigExp(145, 18)
);
```






