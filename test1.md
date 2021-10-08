### Cost Comparison to Deploy DAOS on v1/v2.

**NOTE: All of the gas costs also include creating token for the dao.

Aragon V1 

* Company Template With Agent - 6,203,027
* Company Template Without Agent - 6,053,818
* Membership Template Without Agent - 6,880,665
* Dandellion Template With Agent - 4,911,153 + 4,803,158
* Open Enterprise - 6,139,091 + 4,232,197
* Fundraising - 5,242,885 + 3,348,500 + 6,011,362 + 1,107,911

Govern v2

* With proxies - 1,139,817
* Without Proxies - 9,255,550

Let's calculate this in USD and take the least Gas Cost from V1 and V2. We'll have the following gas costs. `6,053,818 & 1,139,817`. If the `base fee` is 50GWEI(this is just the good amount to take into consideration - as of writing this, it's 113 Though).

* v1 - `6,053,818 * (50 + 1)` = `0.308 ETH` = 1059$
* v2 - `1,139,817 * (50 + 1)` = `0.058 ETH` = 199$

v1 is 5.32 times more expensive, than v2.

**SUM UP 1:** It's a must that we use proxy (EIP-1667) for the Zaragoza if we want to have gas costs cheaper.



### Aragon V1

Aragon V1 consists of the following parts.

* AragonOS (The main smart contracts)
* Dao templates
* Templates' apps

--- 

* Templates App

Let's start with Templates apps and go through all the way of the architecture.

* Aragon Deploys ENS registry where it stores names as keys and values as addresses. Names are `finance.aragonpm.eth`, `voting.aragonpm.eth`, e.t.c.
* Aragon Deploys AMPRegistry which is responsible for creating new `Repo` contracts. Both of these located in `aragonOS` package.

After this, app creator(let's say the creator of `finance` app) does the following;

* builds the finance app, and deploys it to `IPFS` to get CID. (`cidA`)
* deploys finance contract to the blockchain. (`contractA`)
* calls `APMRegistry`'s function and passes `cidA` and `contractA`. This creates new Repo contract and passes `cidA` and `contractA`. So in the end, we got the following situation.

1. ENS registry now holds [`finance.aragonpm.eth` => Repo Contract Address of finance, created above]
2. Repo Contract Address of finance now holds [`contractA` => `cidA`]

* Template

Let's do an example for Company Template...

* Aragon Deploys CompanyTemplate contract an passes `daoFactory`, `ensRegistry` addresses.
* User calls CompanyTemplte's `newInstanceAndToken` which creates new token + new DAO and then calls `dao.newAppAndInstance`. The way `newAppAndInstance` works is it passes `finance.aragonpm.eth` as the namehash, it also goes to ens, searches `finance.aragonpm.eth` to get Repo Contract and from it, it gets `contractA` address. Then, it creates a new `AppProxyUpgradable` that redirects calls to `contractA` with delegate calls.



Brief overview of steps how dao gets created.

1. Aragon Deploys `Dao Factory`, `ENSRegistry`. 
2. Each template gets deployed only once by us. They have a `Dao Factory` contract address in the constructor.
3. User chooses one of the templates to use. It calls [This](https://github.com/aragon/dao-templates/blob/61def4dba465777683b6fa41c4c12b4032efc1f1/templates/reputation/contracts/ReputationTemplate.sol#L38) which creates token, as well as the dao itself. It uses `Dao Factory` address to call dao creation there. The Dao itself is the `Kernel` contract's proxy.

