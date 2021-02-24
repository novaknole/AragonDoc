Depl


## Prerequisits

* Deploy Gover Contracts and Aragon Protocol.



## Test 1 (FAIL)

Calling `newGovernWithoutConfig` with `address(0)` and `false` as the last parameter for `useProxies` fails. The fix has been pushed.

## Test 2 (PASS)

Calling `newGovernWithoutConfig` with other combinations other the one described in Test 1 have been tested. 

Before moving on: 
 * we create `DAO-4` with `newGovernWithoutConfig` (`address(0), true`) .
 * We update the `snapshot-spaces` directory to put `DAO-4` and the token that it should be using. [Link](https://github.com/snapshot-labs/snapshot-spaces/blob/master/spaces/dai-rinkeby/index.json)
 * We update the `snapshot-plugin` to fetch subgraph data from [Link](https://thegraph.com/explorer/subgraph/novaknole/aragon-govern-rinkeby)
 * We deploy a new contract that has `add` and `subtract` function which adds/subtracts 1 from the state variable. Let's call this `TestContract`.

## Test 3 (PASS)

We call `schedule` and pass the new actions which look like this:

```
const calldata = web3.eth.abi.encodeFunctionCall(GovernQueueConfigure, [{
   executionDelay: 60,
   scheduleDeposit:  { token: "0x6817ACB4c166f18cdB98542fdae8ed4961591e94", amount: bigExp(500, 18).toString() },
   challengeDeposit: { token: "0x6817ACB4c166f18cdB98542fdae8ed4961591e94", amount: bigExp(400, 18).toString() },
   resolver: "0xD974ca84751e59a18E98928eb1Ca3472a10a02C2",
   rules: "0x"
}]);
```

Then, we call `execute`. The config gets changed successfully.

## Test 4 (PASS)

We create a new proposal on the snapshots page. We add 2 choices for the proposal. For the first choice, we add the calldata for `add` and for the second choice,
we add calldata for `subtract`. 

After the proposal is done, we call submit on chain, which calls `schedule` function on the govern queue. 

After this, we call `execute` (we remembered the `time` and `nonce` so that we would pass the same container for `execute` because snapshot doesn't provide `execute` functionality on the queue). `execute` also gets succeeded.


## Test 4(PASS)

If `userA` schedules and then executes it by himself:

After scheduling, `userA` successfully loses `scheduleDeposit.amount` from `scheduleDeposit.token`.

After executing, `userA` successfully gains `scheduleDeposit.amount` from `scheduleDeposit.token`.


## Test 5(PASS)

`userA` schedules the container.

`userB` challenges it only when the current term is up to date. If it's not, we call `heartbeat` by hand. This successfully challenges the container and
creates a dispute.

* **Type 1**: we call `draft` in order to move to the next step, but we don't commit, reveal or do anything on the specified dispute at all. After some time, it will end without doing anything. 

We then call `resolve` on the `GovernQueue` which checks what the ruling was returned and if it's 4 or not. Since we didn't commit anything, winning outcome won't be 4, which means it's a rejection for `GovernQueue`. So, the `resolve` will successfully call `settleRejection`.
* **Type 2**: we call `draft`, then we commit the vote `4` , then we reveal it. After some time, dispute ends. We now call `resolve` which successfully calls `_executeApproved`.

For both of the steps, `scheduleDeposit.amount` and `challengeDeposit.amount`  go correctly to the according parties. (NOTE: This has been tested as `challenger` and `scheduler` was the same person).

## Test 6(PASS)

`veto` works fine too.

---

### Problems

**Problem 2**

The `GovernBaseFactory` creates a new queue with the `dummyConfig()`, which means `collaterals` are `0x0000000...000` with 0 amount. With it, the newly created
queue instance has the hash of the `dummyConfig` applied in the storage. 

The problem: someone schedules the container. Since resolver is `0x0000...000`,  no one will be able to challenge it. In order to challenge it, we have to
go through this flow: 

   * we schedule another container, with the new config which also has the valid `resolver`.
   * We execute it by calling `execute` on a queue which automatically executes `exec` on the Govern instance which automatically executes back the `configure` function on the queue. This flow is must because a) only `Govern` can call `configure` on the Queue. b) only `GovernQueue` can call `exec` on the `Govern`.

The above describes what happens when `resolver` is `0x..00` or anything such as invalid. 


Maybe in the schedule function, we somehow check the `config` validity(if the amounts are set and not 0,  if the `resolver` address implements `IArbitrable`) 
and so on.

**NOTE** When deploying officially, note that setting valid `config` object is MUST.


**Problem 3**

Is it normal that I can submit my proposal on-chain multiple times after the proposal is finished from snapshot ?
