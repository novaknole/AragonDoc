This explains how Aragon Protocol Works.


### Questions...


1. Why does this https://github.com/aragon/protocol/blob/f1b3361a160da92b9bb449c0a05dee0c30e41594/packages/evm/contracts/disputes/DisputeManager.sol#L196 use
`_getDisputeAndRound` and not `getDispute` ? passing 0 as `roundId` doesn't make sense, because if dispute exists, `round 0` also exists automatically.

2. Why can't I submit evidence for each round separately and close evidence for each round separately ? it seems like that `closeEvidencePeriod` is only appropriate
for the `0th round`. I guess, `submitEvidence` and `closeEvidencePeriod` only works in the beginning of the dispute when the round is still 0.

3. I think this serves no purpose - https://github.com/aragon/protocol/blob/f1b3361a160da92b9bb449c0a05dee0c30e41594/packages/evm/contracts/voting/CRVoting.sol#L376

4. it seems like votes can vote `2,3 or 4`. How do we know what each one of those represent ?

5. It seems like that first we have to wait until the whole dispute is finished with the final ruling. and then we call `settlePenalties` one time for round 0, then we call `settleReward`, then we call again `settlePenalties` for round 1, and then we call `settleReward`.

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


### STEP 1 

*  `GovernQueue`'s challenge creates a dispute and passes `possibleRulings` as 2.
*  `CreateDispute` creates a dispute and its first round. 
*  
   * The creator of the dispute must pay `_guardiansNumber * (guardianFee + settleFee + draftFee)`. (`_guardiansNumber` is the number of guardians that should be drafted). The fee gets transfered from `msg.sender` to `Treasury` contract.
 
*  Now, someone has to call `draft` so that guardians(people who will vote) can be selected to resolve the dispute somehow.  
   * The `draft` will always be applied to the last round.  
   * Each round has `guardiansNumber` set on it which is the number how many guardians need to be chosen for drafting/voting. If we decide to
apply `3000` for `guardiansNumber`, `draft` function will never succeed because of gas costs. Because of this, protocol makes sure that maximum number of guardians that can be chosen at a time is `maxGuardiansPerDraftBatch`. If we set our `guardiansNumber` to 3000, it means we call the `draft` function multiple times until all of those `3000` are chosen. 
   * After all `guardiansNumber` is chosen, `draft` immediatelly ends which means no one can call it anymore and the status of the dispute changes from `PreDraft` to `Adjudicating`. 
   * Whoever calls `draft` gets `draftFee * draftedGuardians` as the reward on the `Treasury`. `draftedGuardians` doesn't have to be the `guardiansNumber`, as we allow batching.
   * When the `draft` happens, it's possible that the same `guardian` can be chosen multiple times. When this happens, we increase the `weight` for each guardian
by 1. The more the weight, the more will be penalty in case of losing and the more will be reward in case of winning.

* The actual `draft` process is tricky and let's not get into it. Basically, whoever gets chosen, we lock the `penaltyPct` percent of the `minActiveBalance` on their address and they will not be able to `deactivate/unstake` it.

* The next thing that happens is users can start voting(`commiting`).  They can only vote for the numbers as `2,3 or 4`. 3 Options.
* Then, they can start `revealing`.  Let's say `guardianA` was chosen 3 times, `guardianB` was chosen 1 time.  The `guardianA` votes on `2nd` vote, The `guardianB` votes on `3rd` vote. Now, the `2nd` vote will have 3 votes(weight) and `3rd` vote will have 1 vote(weight). The winner outcome is that has most votes(weight).
* After the `revealing` is done, someone can call `createAppeal` to appeal the round's ruling outcome. Maybe someone didn't like the voting results. 
   * If they appeal and the round is not the last one:
    * `GuardiansNumber` becomes whatever it's in the current round multiplied by `appealStepFactor` 
    * we take  `_guardiansNumber * (guardianFee + settleFee + draftFee) * appealCollateralFactor / 10000` from the creator of the appeal.
   * If they appeal and the round is already the 4th one(last one)
    * `GuardiansNumber` becomes the whole active balance divided by the `minActiveBalance` and the result multiplied by 1000 for division rounding.
    * we take `(_guardiansNumber * guardianFee) / FINAL_ROUND_WEIGHT_PRECISION) * finalRoundReduction / 10000 from the creator of the appeal.
 
 
* Then, comes `confirmAppeal`. If nobody confirms the above appeal and enough time gets passed, the dispute and round will be ended and the final ruling
will be whatever the `createAppeal` came up with. So, if someone makes an `appeal`, it's important that we confirm it, otherwise, `createdAppeal` just wins every time.
 * If the round that has to get created is the last one, 
   * the dispute state becomes `Adjucating`, which means there'll be no `drafting` at all.
   * people now can vote on it, but since there was no draft and there was no choosen guardians, in this last round, anyone can vote if they have enough weight - `(activeBalance * 1000 / minActiveBalance) != 0 `. When they vote, we immediatelly take `activeBalance * penaltyPct / 10000` from their active balance.
 * If the round is not the last one:
   * the dispute state becomes `PRE_DRAFT` , which means we can call `draft` and start choosing guardians as we were doing before. Everything above just gets repeated.
     

