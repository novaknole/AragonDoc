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

### Step 2..

Aragon OS is the package where all the main contracts are stored.. (DaoFactory, Kernel, ACL, .e.t.c)

* So, logically, we need to deploy DaoFacory and that should be it in terms of contracts, because DaoFactory will already include 
other contracts that are necessary for the contract deployment.

* We also need to deploy ENS(if we already got one, which is the case for mainnet/rinkeby/ropsten, then we don’t deploy it and we just pass ENS when we need it, 
otherwise, we deploy our own ENS which is the case in Polygon Mumbai, as this network doesn’t have ENS or whatever the fuck)…

* We also need to deploy APM (Aragon Package Manager). We will see later what it is..

To do those STEP 2 stuff,   we need to do the following, but a little history before then..

In AragonOS, there was APM included, but devs decided to separate it in its own REPO.  Currently, AragonOS doesn’t contain APM and its scripts that we need, but at some previous time, it did.  The best way would be to do the below 3 commands on APM package which is separated, but somehow, the same 3 commands don’t work in APM package. God knows why. So I am gonna do the following. In AragonOs, we will move to master branch instead of default `next` branch. Master branch contains v4.4.0 latest and it’s really a stable, good version. After v4.4.0, devs did more features, but we don’t need disputable feature for now. :) NOTE: would be good if we were doing this in APM package, but as I said, somehow it doesn’t work. You might make it work though. 

a. let’s clone this https://github.com/aragon/aragonOS. 
b. Run `yarn`
c. Run `yarn compile`
D  Now, note that we use Aragon OS v4.4.0 which contains the fixes for ACL , I don’t think main net contracts on ethereum had this fix as it was found later.  See https://github.com/aragon/aragonOS/releases v4.4.0 explanation.

1. OWNER=0x94C34FB5025e054B24398220CBDaBE901bd8eE5e npx truffle exec --network mumbai scripts/deploy-test-ens.js (Deploys ENS) 
2. OWNER=0x94C34FB5025e054B24398220CBDaBE901bd8eE5e npx truffle exec --network mumbai scripts/deploy-daofactory.js (Deploys DaoFactory) 
3. OWNER=0x94C34FB5025e054B24398220CBDaBE901bd8eE5e ENS=0x431f0eed904590b176f9ff8c36a1c4ff0ee9b982 DAO_FACTORY=0xa48e321d8ebab7ccd52503d630c894b62a2f639b npx truffle exec --network mumbai scripts/deploy-apm.js 

### Step 3...(Deploying aragon-apps)

**IMPORTANT** The below won't work work for `voting-disputable` and `agreement` apps.

* git clone https://github.com/aragon/aragon-apps
* create `.env` file in the root directory of the whole package and add `ETH_KEYS=YourPrivateKey`. Make sure it starts with `0x`.

Now, let's deploy finance as an example and repeat the same steps for each one of them.

* go to `buidler.config.js` and add 

```
 mumbai: {
   url: 'https://polygon-mumbai.g.alchemy.com/v2/z3go4SKtSuiegUwtfkfd5tBCLDTcwYP_',
   accounts: ["yourPrivateKey"]
 }
```

For your own network, act accordingly and add specific url.

* go to `app` folder of the finance folder, and build it with `npm run build`.
* the previous creates the `build` folder inside `app` folder. Copy its contents and place it inside `dist` folder of finance folder.
* update `arapp.json` and add this
```
     "mumbai": {
      "registry": "0xc38f78e76869116c0e8ba3d8bcdf5887765287fa", // (this is ens registry)
      "appName": "finance.aragonpm.eth", // app name
      "network": "mumbai"
    }
```
* run `npx buidler publish major --network mumbai --ipfs-api-url https://ipfs.infura.io:5001` . `major` is not necessary unless you're updating the contract code as well. So make sure to pass `minor` or `patch`.

### Step 4...(Deploying dao templates)

**NOTE 1** The dao templates package currently only contains the necessary code for company, reputation, membership templates
**NOTE 2** Each template contains `truffle.js` file which imports `@aragon/truffle-config-v4`. If that package doesn't contain the chain that you want, let us quickly know and we will update the package to include your new chain as well in seconds.

We came at this step so that we still need `MiniMeTokenFactory` and `AragonID` contracts deployed.. 

* create `.env` file in the root directory of the whole package and add `ETH_KEYS=YourPrivateKey`. Make sure it starts with `0x`.

Now, I am gonna do the deployment for `company template`. Same should apply to others.

* go to `cd shared` and run `yarn link`. Then go to each template you want to deploy and run `yarn link @aragon/templates-shared`.
* go to `cd templates/company` and run `yarn` to install dependencies.
* update `arapp.json` for each of the template. Pay attention to the `appName` and `registry` addresses. You should know how by now, but anyways, you gotta add the same kind of thing for your network. In this case, As an example, I'd add this:
```js
"mumbai": {
   "appName": "company-template.aragonpm.eth", // template name 
   "network": "mumbai", // network name..
   "registry": "0x431f0eed904590b176f9ff8c36a1c4ff0ee9b982", // ens registry address
   "wsRPC": "wss://matic-testnet-archive-ws.bwarelabs.com"
}
```
* run `yarn compile` 
* In the package.json, you will see the below. Make sure to change `ens` and `dao-factory` with what you got in the previous steps.

```js
"deploy:mumbai": "truffle exec ./scripts/deploy.js --network mumbai --ens yourENSAddress --dao-factory yourDaoFactoryAddress"
```

and run this with `yarn deploy:mumbai`. If you're deploying this on another network, just try to change `--network` argument.


If you follow all the previous steps, this will deploy minime token factory and aragonId only + This will also deploy the company template with all the arguments. Copy those addresses. **IMPORTANT** Now, if you decide to deploy other templates, make sure to check their package.json, change `ens` and `dao-factory` addresses and also add another argument `--minime-token-factory minime-token-factory-address-here`. adding another argument is necessary, because we don't need to deploy minime token each time.

Now,  we gotta publish the contract and ipfs content to APM and ipfs.

```js
npx buidler compile && npx buidler publish major --contract templateAddressYouGotInPreviousStep --network mumbai --ipfs-api-url https://ipfs.infura.io:5001`
```

And we got the company template as well.


# Some Learnings

### Template Stuff

* Aragon Client has templates(company, membership, reputation, e.t.c).. They are located in dao-templates repository.
 They consist of arapp.json, manifest.json, their respective contract. for Company, it's CompanyTemplate.sol, for reputation, it's ReputationTemplate.sol.
Steps that need to be done.. Let's take company template as an example.
  * we deploy the company's directory to the ipfs. = ipfsHash (This deploys `artifact.json, manifest.json, code.sol` . `CompanyTemplate.sol` becomes `code.sol`.  `arapp.json` becomes `artifact.json`)
  * we deploy `CompanyTemplate.sol` = contractAddress
  * When we deploy the template for the first time, It creates a `Repo` contract for it through `RepoRegistry` contract's `newRepoWithVersion` function. `newRepoWithVersion` expects these arguments: `company-template.aragonpm.eth`, [1.0.0], `contractAddress`, `contentURI(ipfsHash)`. 
  * `Repo` is basically where it contains all the versions of contract and ipfs stuff for these templates. If we wanna update the template, we update it, publish it to `Repo`'s `newVersion` function. And client then can fetch the latest `contractAddress` and `ipfs` stuff.
  * After the `Repo` contract address is created, it stores this Repo address on the ENS registry with the key of template `company-template.aragonpm.eth`.

  Now when the client needs the companyTemplate contract, it calls ENS first with the key `company-template.aragonpm.eth` to get the repo address.
When it has the repoAddress, it calls getLatest() on this repoAddress, and we get [1.0.0, contractAddress, ipfsHash]. if we updated the version, for sure
we would get different than 1.0.0. 

**NOTE** Aragon Deploys those templates. Users don't need to deploy them... 

### Aragon App

  Each template needs some apps(voting, finance, and many more.), otherwise, it's a template, and nothing more. Now,  we also need to be doingthe same kind of deployments for these apps.

* Each app has its own name(voting.aragonpm.eth, finance.aragonpm.eth, .e.t.c) These apps are located in Aragon apps.
* They are registered on ENS with their name and corresponding address is the repo addresses. They have their own repo contract. https://github.com/aragon/apm/blob/0d68cd2bf4e4e22c8871614b93a87e82e7586515/contracts/apm/Repo.sol. Repo contract for each one of the app is created by the app developer itself. Users don't deploy this repo address. Client just fetches the corresponding version
* If we update the contract on an app, we have to pass `major` in buidler publish. When this all happens, the UI on client will still use the old version. If we want it to use new versions, we go to client and upgrade app from there which causes new action to be performed and it needs voting to be passed. If we don't update contract and just UI on an app, we publish it with `minor/patch` and if that happens, UI is going to reflect it automatically. 


**NOTE** When we deploy the aragon app or template with `npx buidler publish`, we can pass manager address.. This manager address is basically who can update versions of those apps or template. By default, it's the deployer's address.


### Summary of Aragon Contracts.

Aragon DAO is presumed to be called `Kernel` Contract. 

* Aragon Deploys `Kernel` contract as the base contract one time only.
* When User deploys dao, we deploy `KernelProxy` contract and pass the base contract s that this proxy uses the base as the delegate calls.
* If user chooses company template and deploys it with let's say voting and finance contract, what happens is we deploy `AppProxyUpgradable` contracts for each one of them(voting, finance). It's a proxy contract and while we deploy it, we also call `initialize()` function on both voting and finance so that state can be initialized on Proxy contracts as well.


### Let's go more advanced. 

* Aragon Deploys `ACL` and `Kernel` contracts one time. Let's call those `base contracts(base ACL and base Kernel)`. Then deploys `DaoFactory` and passes those base contracts.
* DaoFactory creates new DAOS. The new dao is a `KernelProxy` that will redirect all calls to `base kernel` through delegate call. We also do the same shit for `base acl` and get it as `KernelProxy`. We then set this new `ACL` address on this dao.

So basically, We have a Kernel that has `ACL` inside.

* We then deploy `Company template` one time with the `DaoFactory` Address.

Let's say user now wants to create a dao.

* Company Template creates a new dao through `DaoFactory.newDao`. Which means at this step, we got Kernel and ACL inside Kernel.
* We then install new apps. For each app, `AppProxyUpgradable` contract gets created and we set the same ACL that Kernel had on each app. **IMPORTANT**: If we want to change roles/permissions on any of the apps, we gotta go through Kernel's ACL..
