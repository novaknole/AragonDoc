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

* git clone https://github.com/aragon/aragon-apps

Now, let's deploy finance as an example and repeat the same steps for each one of them.

* go to finance folder and `yarn add -dev @aragon/buidler-aragon`
* go to `buidler.config.js` and add `usePlugin('@aragon/buidler-aragon')`
* go to `buidler.config.js` and add 

```
 mumbai: {
   url: 'https://polygon-mumbai.g.alchemy.com/v2/z3go4SKtSuiegUwtfkfd5tBCLDTcwYP_',
   accounts: ["yourPrivateKey"]
 }
```

* go to `app` folder, and build it with `npm run build`.
* the previous creates the `build` folder inside `app` folder. Copy its contents and place it inside `dist` folder of finance folder.
* update `arapp.json` and add this
```
     "mumbai": {
      "registry": "0xc38f78e76869116c0e8ba3d8bcdf5887765287fa", // (this is ens registry)
      "appName": "finance.aragonpm.eth",
      "network": "mumbai"
    }
```
* run `npx buidler publish major --network mumbai --ipfs-api-url https://ipfs.infura.io:5001`

### Step 4...(Deploying dao templates)

We came at this step so that we still need `MiniMeTokenFactory` and `AragonID` contracts deployed.. 

* go to `cd shared/lib/TemplatesDeployer.js`
* We have to comment some stuff. 

1. find this `await this._checkAppsDeployment()` and comment it out.
2. where we call `registerDeploy` function, comment it.

* go to `cd shared/scripts/deploy-template.js` and make `verbose:true` instead of false.

We comment these stuff, because if we don't, it tries to deploy contract and also publish it to APM. That would be good, but problem is it tries to publish
to APM with `0x` empty hash as ContentURI(ipfs hash) and this brings whole other problems. So won't go into deep details for now.

Now, I am gonna do the deployment for `company template`. Same should apply to others.

* update `arapp.json` . You should know how by now. hahaha.
* add this to `package.json` in company template.

```js
"deploy:mumbai": "truffle exec ./scripts/deploy.js --network mumbai --ens 0x431f0eed904590b176f9ff8c36a1c4ff0ee9b982 --dao-factory 0xa48e321d8ebab7ccd52503d630c894b62a2f639b"
```

and run this with `yarn deploy:mumbai`.


If you follow all the previous steps, this will deploy minime token factory and aragonId only + This will also deploy the company template with all the arguments. Copy those addresses. If you decide to update company template, You gotta update the above `deploy:mumbai` script and also pass `minime-token-factory` and `aragonId`..


Now,  we gotta publish the contract and ipfs content to APM and ipfs.

1. copy the same buidler settings from `aragon-apps` package into `company` template.
2. run 

```js
npx buidler compile && npx buidler publish major --contract 0x282d6bf70d08bc02781631678111e772994a3d79 --network mumbai --ipfs-api-url https://ipfs.infura.io:5001`
```


And we got the company template as well.