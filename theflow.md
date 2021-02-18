Deploys....


**Problem 1**

Let's run 

```js
baseFactory.methods.newGovernWithoutConfig("GIORGI-3", "0x0000000000000000000000000000000000000000", "GIORGI1", "GIO", false).send({
```
This means that baseFactory has to create new DAO on the name `GIORGI-3`. We pass `address(0)` for the token(second argument), so that it would also create
the token for us.... The problem is if we pass `address(0)` as the second argument and `false` as the last argument.

Seems like token creation fails with `revert - acl:auth`.  passing `true` solves the problem, but it should be working with `false` too. 

The problem arises here : [Link](https://github.com/aragon/govern/blob/5c0293fda66c188b971f96de0666b6309e379c78/packages/govern-token/contracts/GovernTokenFactory.sol#L58)

As shown, `GovernTokenFactory` calls `mint` on the `minter`, but `GovernTokenFactory` doesn't have the rights. The only person who has this right
is `GovernBaseFactory`. 

Let's look at this: [Link](https://github.com/aragon/govern/blob/5c0293fda66c188b971f96de0666b6309e379c78/packages/govern-token/contracts/GovernTokenFactory.sol#L105)

The idea is that the second argument should be `address(this)` instead of `address(_initialMinter)`. The problem doesn't show when we pass `true` as the last
argument, because that line passes `address(this)` correctly. [Link](https://github.com/aragon/govern/blob/5c0293fda66c188b971f96de0666b6309e379c78/packages/govern-token/contracts/GovernTokenFactory.sol#L52)


**Problem 2**

The `GovernBaseFactory` creates a new queue with the `dummyConfig()`, which means `collaterals` are `0x0000000...000` with 0 amount. With it, the newly created
queue instance has the hash of the `dummyConfig` applied in the storage. 

Now, I can still call `schedule`, pass the same config (since it's important that the config we pass matches the current `configHash`). Since the queue was created with dummyConfig, Then, I can pass the same dummyConfig and it will let me schedule the event. Now, the problem is that `resolver` is `0x...00`, which means that we will never be able to challenge this.

Example:

```js

const from = (await web3.eth.getAccounts())[0];
const nonce = (BigNumber.from(1));
const currentDate = Math.round(Date.now() / 1000) + 0 + 60;
const FAILURE_MAP ='0x0000000000000000000000000000000000000000000000000000000000000000';
const EMPTY_BYTES = '0x00';
  
  
governQueue.methods.schedule(
        {
          payload: {
            nonce: nonce.toString(),
            executionTime: currentDate,
            submitter: "0x94C34FB5025e054B24398220CBDaBE901bd8eE5e",
            executor: "0x94C34FB5025e054B24398220CBDaBE901bd8eE5e",
            actions: [],
            allowFailuresMap: FAILURE_MAP,
            // proof in snapshot's case, could be the proposal's IPFS CID
            proof: EMPTY_BYTES
          },
          config: {
            executionDelay: 0,
            scheduleDeposit: {
              token: "0x0000000000000000000000000000000000000000",
              amount: 0
            },
            challengeDeposit: {
              token: "0x0000000000000000000000000000000000000000",
              amount: 0
            },
            resolver: "0x0000000000000000000000000000000000000000",
            rules: "0x"
          }
        }
)
```

The solution could be that we somehow do some checks on the `schedule` function so that at least resolver has specific `implementsInterface` and token addresses
are valid. This would make sure that we wouldn't allow scheduling with dummyConfig and make users run `setConfig` with the actual, valid config.



