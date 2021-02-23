This explains how Aragon Protocol Works.


### Questions...


1. Why does this https://github.com/aragon/protocol/blob/f1b3361a160da92b9bb449c0a05dee0c30e41594/packages/evm/contracts/disputes/DisputeManager.sol#L196 use
`_getDisputeAndRound` and not `getDispute` ? passing 0 as `roundId` doesn't make sense, because if dispute exists, `round 0` also exists automatically.

2. Why can't I submit evidence for each round separately and close evidence for each round separately ? it seems like that `closeEvidencePeriod` is only appropriate
for the `0th round`. I guess, `submitEvidence` and `closeEvidencePeriod` only works in the beginning of the dispute when the round is still 0.

3. I think this serves no purpose - https://github.com/aragon/protocol/blob/f1b3361a160da92b9bb449c0a05dee0c30e41594/packages/evm/contracts/voting/CRVoting.sol#L376

4. it seems like votes can vote `2,3 or 4`. How do we know what each one of those represent ?

### Guardians Registry

Each guardian has:

* Active Balance (This uses `Tree` structure)
* AvailableBalance
* LockedBalance
* ActivationLocks
  * total
  * lockedBy[lockManager] (this returns amount)
  

* AvailableBalance is just the balance that user has and he can anytime withdraw this money with `unstake` function. If he wants to deposit money, he calls `stake`.
* As we know, after creating a dispute, people have to vote so that they decide how to resolve the dispute. The people that we choose are called guardians and
they should already have deposited money with `stake` function if they want to be chosen, but this is not enough. They also should have called `activate`, so that 
we move their money from their `availableBalance` to `ActiveBalance`.
* So, if they have done the above steps, they can be choosen as guardians(people who can vote). While they get chosen, we update their `lockedBalance`, and they
can't withdraw their `lockedBalance` anymore. 
* updating `ActivationLocks` happen with `LockActivation`. Let's explain why this exists. Even if users have activated their balance, they can anytime deactivate
it with `deactivate` function, if the amount they want to deactivate is not already in `LockedBalance`.  Also, users can also call `lockActivation` and this will
make sure that if they call `deactivate`, they will only be able to deactivate the amount so that remaining ones will be >= their `activationLocks` amount. 
Basically this means that users can't deactivate the `activationLocks` money, unless they call `unlockActivation`.

### First Step (We create the dispute)


### STEP 1 (Someone creates a dispute)
