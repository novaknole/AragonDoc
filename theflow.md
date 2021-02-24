Depl



| Test                                                                                                                                   | slot 1 | slot 2 |   |   |   |   |   |   |
|----------------------------------------------------------------------------------------------------------------------------------------|--------|--------|---|---|---|---|---|---|
| `baseFactory.methods.newGovernWithoutConfig( "DAO-7" ,  "0x0000000000000000000000000000000000000000" ,  "GIORGI1" ,  "GIO" ,  false fffffffffffffffffffffff)` | `nice` | test3  |   |   |   |   |   |   |
|                                                                                                                                        |        |        |   |   |   |   |   |   |
|                                                                                                                                        |        |        |   |   |   |   |   |   |
|                                                                                                                                        |        |        |   |   |   |   |   |   |



















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

The problem: someone schedules the container. Since resolver is `0x0000...000`,  no one will be able to challenge it. In order to challenge it, we have to
go through this flow: 

   * we schedule another container, with the new config which also has the valid `resolver`.
   * We execute it by calling `execute` on a queue which automatically executes `exec` on the Govern instance which automatically executes back the `configure` function on the queue. This flow is must because a) only `Govern` can call `configure` on the Queue. b) only `GovernQueue` can call `exec` on the `Govern`.

The above describes what happens when `resolver` is `0x..00` or anything such as invalid. The another scenario happens when we don't change `config` at all
and start to schedule containers. With this, the amount on `scheduleDeposit` and `challengeDeposit` will be 0, which means anyone can schedule/challenge
without any penalties.


Maybe in the schedule function, we somehow check the `config` validity(if the amounts are set and not 0,  if the `resolver` address implements `IArbitrable`) 
and so on.

**NOTE** When deploying officially, note that setting valid `config` object is MUST.


**Problem 3**

Is it normal that I can submit my proposal on-chain multiple times after the proposal is finished from snapshot ?

**Problem 4** 

Let's say we deployed Aragon Protocol. and `START_DATE` was specified as 2021, 15th February and `TERM_DURATION` as 60*10 seconds. Let's say 5 days have passed and we didn't touch protocol at all.(This means that we never called `ensureCurrentTermId` which would call `_heartbeat` automatically). Now, when we call `createDispute` from the `GovernQueue` , it will fail due to `CLK_TOO_MANY_TRANSITIONS`. The problem is that the transition count returned from `_neededTermTransitions` is more than 1. This means that we have to call `heartbeat` by hand. Now, with the DURATION with 60*10 seconds, and 5 days have passed since `START_DATE`,  It appears that I have to call `heartbeat` 558 so that it gets up to date. 558 is so big that Because of this, the gas usage is too huge, it almost requires the same amount as the block gas limit which means my transaction most of the time will never get executed.


**Problem 5**

```js
//Govern subgraph
export function buildEventHandlerId(
  containerHash: string,
  eventName: string,
  logIndex: string
): string {
  return containerHash +  eventName + logIndex.toString()
}
```


// Subgraph instance failed to run: Error while processing block stream for a subgraph: tried to set entity of type `ContainerEventExecute` 
// with ID "0x8942a520ad1d0965f25fe3f5bcae79c841da5ed4047ce35575c2b89bad1da1fe0x9" but an entity of type `ContainerEventSchedule`, 
// which has an interface in common with `ContainerEventExecute`, exists with the same ID, code: SubgraphSyncingFailure, id: QmPLjMDYPBCyMjDNiSbYG4ZeTgqFUxxk5R6SvF6aEYZEWk


**Problem 6**

Subgraph instance failed to run: Failed to process trigger in block #8117766 (cacdfd5cd0f8a1ace6d0fe41d5063ce4d32f80ecee7d26aeb238a4a78599a941), transaction 4b32577a9efe23a77704e7dcb54d90d8e0e3332b12fc28742e083114c08f5a87: Entity ContainerEventChallenge[0x3ec3929f0af90f2d66b5bc0286f87429ac659d411582babe9076684e30c49c60challenge0xf]: missing value for non-nullable field `resolver` wasm backtrace: 0: 0x225c - <unknown>!generated/schema/ContainerEventChallenge#save 1: 0x22fc - <unknown>!src/utils/events/handleContainerEventChallenge 2: 0x2462 - <unknown>!src/GovernQueue/handleChallenged , code: SubgraphSyncingFailure, id:
  
```js
handleChallenged() 

let resolver = ConfigEntity.load(queue.config).resolver
let containerEvent = handleContainerEventChallenge(container, event, resolver)

export function handleContainerEventChallenge(
  container: ContainerEntity,
  ethereumEvent: ChallengedEvent,
  resolver: Bytes
):
containerEvent.collateral = collateral.id
containerEvent.resolver = resolver
```

