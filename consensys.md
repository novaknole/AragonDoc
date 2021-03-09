https://consensys.net/diligence/audits/private/fqj3znl1xk6iwr#gas-dos-in-governqueue-schedule-allows-anyone-to-create-unchallengable-containers

After reading the description about `4.1 Gas DoS in GovernQueue.schedule allows anyone to create unchallengable Containers Major`, I will explain again what we think
the solutions could be.

`GovernQueue` contract https://github.com/aragon/govern/blob/master/packages/govern-core/contracts/pipelines/GovernQueue.sol


Just to briefly explain what the audit about 4.1 states is that It's possible that someone calls `schedule` function on the `GovernQueue` and passes the container
struct, and after this, that someone or anyone else decides to call `challenge` function on the `GovernQueue` again. The problem is `challenge` function
might get failed for the `scheduled container`, meaning that we will never be able to challenge it. This is called `DOS attack`.

This problem happens when the passed container to the `schedule` function is large. `schedule` function has small code and doesn't use much, so it still 
allows us to successfully call it, but `challenge` function uses more advanced things and things that cause copying into memory over and over again. 
This results in much larger gas usage and in the end, it almost goes above the block gas limit. for small containers, this problem doesn't happen, 
because block gas limit currently is 15,000,000 and even if `challenge` function has big code and causes memory copying many times, 
it still doesn't exceed the 15,000,000 limit.


**Solution 1**: We can do as the audit suggests. For the `challenge` function, we only pass the hash of the container which means that copying from memory into
memory or copying from calldata into memory for huge container wouldn't happen anymore. This solves the problem, but brings another. If we look closely here:
[Link](https://github.com/aragon/govern/blob/2c1cad2bc65c713978d895cb222a1eb5bf7bc7b0/packages/govern-core/contracts/pipelines/GovernQueue.sol#L175), what this
line does is we give the whole container to the arbitrator(the thing that takes care about how the voting will go, e.t.c). If we only pass `hash` in the `challenge`
function, we will not be able to pass the whole container to the arbitrator and that's not good, because `arbitrator` needs to have this whole container data due 
to a couple of reasons:
* It emits the whole container. This helps subgraphs because someone listens to arbitrator's graph specifically and waits for the new disputes again and again.
* Who knows, maybe that container is needed on-chain in the arbitrator for some reason. 

Due to the above reasons, we don't consider `Solution 1` as acceptable yet.

Before we explain the other solutions, let's quickly think about why passing huge containers could cause `challenge` to fail. If we look at the container's 
structure found in [here](https://github.com/aragon/govern/blob/master/packages/erc3k/contracts/ERC3000Data.sol), it's easy to note 
what could cause huge containers. We have `bytes rules, bytes proof, Actions[] Action, and bytes data in each Action`.(They are all dynamic).
Other parameters are pre-defined length and they are so small. So, those 3 dynamic parameters|things are the problem.

**Solution 2**:  One of the solution that we can think about is that before we call `schedule`, we add `rules`, `proof`, `Actions[] Action` to the ipfs which
gives us the according hashes and that's what we put in the container. So `bytes rules` will be changed to `bytes32 rules` that will store the hash of the ipfs.
Same things for Actions and proof. The problem with this approach is that it's very complex. Here is why: 
* before a customer calls schedules, it calls ipfs to store those data mentioned above, then schedules the container. 
* When someone challenges it, we would pass the whole container to the arbitrator, but `rules,proof, Actions` would be the ipfs hashes 
which mean that arbitrator right away would not be able to use `Actions[]` for example in the `createDispute`. 
* if someone listens to the Arbitrator with the help of subgraph, that emits the events, he/she would have to go to ipfs to grab the actual data(rules, proof, Actions)
* **Main problem**: Somehow, along the way, in the `resolve` function, it's still mandatory to pass the array of `Action`, because if we need to execute it,
we should be passing all the `Action[]` to the executor [Link](https://github.com/aragon/govern/blob/2c1cad2bc65c713978d895cb222a1eb5bf7bc7b0/packages/govern-core/contracts/pipelines/GovernQueue.sol#L304)
This somehow makes me believe that the DOS attack could be the same. Someone still could make `schedule` function successful and then, `resolve` function
would never be able to finish due to its gas usage exceeding the block gas limit. Though, I think this could be easily possible, I also think that it couldn't
be that easy. This is because `resolve` function doesn't seem to be using lots of gas and even passing huge `Actions[]` data, it could still work as long as
`schedule` function succeeds.

**Solution 3**: We directly restrict sizes of those dynamic parameters in the container. Let's say `rules` can be 100 length, same for `proof` and
for Actions[], the maximum array length could be 100, and each `bytes data` in it could be 100 length size maximum. This is also a problem because customers
are dependent on the restricted sizes and what if their requirements are like huge `rules, proof` or `Action[]`.


**Solution 4**: Maybe we could store `rules` and `proof` in the ipfs and we still continue passing `Action[]` in the container directly. but the catch is that
we somehow should make sure that we restrict the size of Action[] (let's say 100) and we also restrict the size of each `bytes data` in it to let's say 100.
This solution just makes sure that customers can still use huge `rules` and `proof`. If they want to schedule container with more Actions then 100, then they
schedule multiple containers.






