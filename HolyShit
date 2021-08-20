# Let’s Fucking Go…

### STEP 1

`nano ~/.aragon/mumbai_key.json`.

Include something like this shit in it.

```
{
  "rpc": "https://polygon-mumbai.g.alchemy.com/v2/z3go4SKtSuiegUwtfkfd5tBCLDTcwYP_",
  "keys": ["PrivateKey"]
}
```

### Step 2... 

This step is something that mightn't need to be done, but sometimes it requires. So, let's anyways do it.

* clone dao-templates package and in the root, run `npm install @aragon/cli`.
* go to `cd node_modules/@aragon/cli/dist/apm_cmds/util/runPublishTask.js` and add `transaction.chainId = 80001` or `137`. whatever polygon you're using.
* go to `cd node_modules/@aragon/cli/dist/lib/deploy/deploy.js` and `chainId: 80001 or chainId: 137` in the `send`.
* go to `cd node_modules/@aragon/apm/index.js` and update this line `ipfs: ipfs(options.ipfs),` with `ipfs: ipfs(api)`,


### STEP 2

Aragon OS is the package where all the main contracts are stored.. (DaoFactory, Kernel, ACL, .e.t.c)

a. So, logically, we need to deploy DaoFacory and that should be it in terms of contracts, because DaoFactory will already include 
other contracts that are necessary for the contract deployment.
b. We also need to deploy ENS(if we already got one, which is the case for mainnet/rinkeby/ropsten, then we don’t deploy it and we just pass ENS when we need it, 
otherwise, we deploy our own ENS which is the case in Polygon Mumbai, as this network doesn’t have ENS or whatever the fuck)…
c.  We also need to deploy APM (Aragon Package Manager). We will see later what it is..

To do those STEP 2 stuff,   we need to do the following, but a little history before then..

In AragonOS, there was APM included, but devs decided to separate it in its own REPO.  Currently, AragonOS doesn’t contain APM and its scripts that we need, but at some previous time, it did.  The best way would be to do the below 3 commands on APM package which is separated, but somehow, the same 3 commands don’t work in APM package. God knows why. So I am gonna do the following. In AragonOs, we will move to master branch instead of default `next` branch. Master branch contains v4.4.0 latest and it’s really a stable, good version. After v4.4.0, devs did more features, but we don’t need disputable feature for now. :) NOTE: would be good if we were doing this in APM package, but as I said, somehow it doesn’t work. You might make it work though. 

a. let’s clone this https://github.com/aragon/aragonOS. 
b. Run `yarn`
c. Run `yarn compile`
D  Now, note that we use Aragon OS v4.4.0 which contains the fixes for ACL , I don’t think main net contracts on ethereum had this fix as it was found later.  See https://github.com/aragon/aragonOS/releases v4.4.0 explanation.

1. OWNER=0x94C34FB5025e054B24398220CBDaBE901bd8eE5e npx truffle exec --network mumbai scripts/deploy-test-ens.js (Deploys ENS) 
2. OWNER=0x94C34FB5025e054B24398220CBDaBE901bd8eE5e npx truffle exec --network mumbai scripts/deploy-daofactory.js (Deploys DaoFactory) 
3. OWNER=0x94C34FB5025e054B24398220CBDaBE901bd8eE5e ENS={ens} DAO_FACTORY={daofactory} npx truffle exec --network mumbai scripts/deploy-apm.js 

### Step 3

Now, we need to deploy MiniMeTokenFactory and AragonID (we can find better way maybe ? )

This is tricky, as I am not sure if this deploys the latest artifacts from those contracts. **NOTE** that the way below could also deploy all the stuff that we did
above, but I still prefer to use the above one as I am more sure that it uses latest artifacts.

a. Clone https://github.com/aragon/dao-templates
b. Run `yarn`
c. The template that you are planning on publishing, go to that package,  and whatever is there, change it with 

```js
module.exports = require('@aragon/truffle-config-v4’)
```

We do this because truffle-config-v4 already contains Mumbai and matic network.
		
* cd templates/company. (Anyways run `yarn` again )
* Run `yarn compile`
* Let's uncomment everything from the `TemplatesDeployer` so that it only deploys aragonID and MiniMeTokenFactory. 
* add this in package.json and run it 
```
"deploy:mumbai": "truffle exec ./scripts/deploy.js --network mumbai --ens 0x10e4c5975f2c2fc8cc64ba6b334a7ba5675082da --dao-factory 0x266817b8cc1a466101355cdf2fa18211a745f15f
```
* Above gives us aragonId and MinimeTokenFactory.. Now let's change the command to this:

### Step 4 (Deploying Aragon Apps)


### Step 5 (Deploying Templates)

Let's do this for Company Template.

* update package.json such as

```
"deploy:mumbai": "truffle exec ./scripts/deploy.js --network mumbai --ens 0x67be86e17d43ed284c008ce0e614bd0b69497869 --dao-factory 0x266817b8cc1a466101355cdf2fa18211a745f15f --mini-me-factory 0x51157EF91f7848105a6be6d6d9A5933365184c34 --aragon-iD 0xf5042ec5888d404b7851db02fc56317818fcf88f"
```

* Make sure that above gives us template address. You can check the code of TemplatesDeployer to achieve this... Also, note that we should not register any package in it
* Then, we can run

```

===== STEP 4 =========



====== STEP 4 =========

DEPLOYING the template… Let’s goo

1. Cd templates/company
2. Npm install @aragon/cli (globally installed Aragon cli has some problem)
3. Go to `arapp.json` and put this
"mumbai": {
      "appName": "company-template.aragonpm.eth",
      "network": "mumbai",
      "registry": “0x67be86e17d43ed284c008ce0e614bd0b69497869”,
      "wsRPC": "wss://matic-testnet-archive-ws.bwarelabs.com"
    }




