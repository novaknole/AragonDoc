This explains how Aragon Protocol Works.

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

**NOTE:**
 * `FINAL_ROUND_WEIGHT_PRECISION` exists to ensure no rounding division problem exists.

*  `GovernQueue`'s challenge creates a dispute and passes `possibleRulings` as 2.
*  `CreateDispute` creates a dispute and its first round. 
*  
   * The creator of the dispute must pay `_guardiansNumber * (guardianFee + settleFee + draftFee)`. (`_guardiansNumber` is the number of guardians that should be drafted). We pass this number, when we deploy the protocol. The fee gets transfered from `msg.sender` to `Treasury` contract.
 
*  Now, someone has to call `draft` so that guardians(people who will vote) can be selected to resolve the dispute somehow.  
   * The `draft` will always be applied to the current round.  
   * Each round has `guardiansNumber` set on it which is the number how many guardians need to be chosen for drafting/voting.(Remember, this is the number we specified in the config, while we deployed the protocol). If we decide to apply `3000` for `guardiansNumber`, `draft` function will never succeed because of gas costs. Because of this, protocol makes sure that maximum number of guardians that can be chosen at a time is `maxGuardiansPerDraftBatch = 81`. If we set our `guardiansNumber` to 3000, it means we must call the `draft` function multiple times until all of those `3000` are chosen. 
   * After all `guardiansNumber` is chosen, `draft` immediatelly ends which means no one can call it anymore and the status of the dispute changes from `PreDraft` to `Adjudicating`. 
   * Whoever called `draft` gets `draftFee * draftedGuardians` as the reward on the `Treasury`. `draftedGuardians` doesn't have to be the `guardiansNumber`, as we allow batching. 
   * When the `draft` happens, it's possible that the same `guardian` can be chosen multiple times. When this happens, we increase the `weight` for each guardian
by 1. The more the weight, the more will be penalty in case of losing and the more will be reward in case of winning.

* The actual `draft` process is tricky and let's not get into it. Basically, whoever gets chosen, we lock the `penaltyPct` percent of the `minActiveBalance` on their address and they will not be able to `deactivate/unstake` it.

* The next thing that happens is users can start voting(`commiting`).  They can only vote for the numbers as `2,3 or 4`. 3 Options.
* Then, they can start `revealing`.  Let's say `guardianA` was chosen 3 times, `guardianB` was chosen 1 time.  The `guardianA` votes on `2nd` vote, The `guardianB` votes on `3rd` vote. Now, the `2nd` vote will have 3 votes(weight) and `3rd` vote will have 1 vote(weight). The winner outcome is the one that has most votes(weight).
* After the `revealing` is done, someone can call `createAppeal` to appeal the round's ruling outcome. Maybe someone didn't like the voting results. 
   * If they appeal and the round is not the last one:
    * `GuardiansNumber`(the number of guardians that have to be chosen for the next round's draft) becomes whatever it's in the current round multiplied by `appealStepFactor` 
    * we take  `_guardiansNumber * (guardianFee + settleFee + draftFee) * appealCollateralFactor / 10000` from the creator of the appeal. (10000 is just there to ensure no problem with rounding, basically `appealCollateralFactor` will also be in the same range , let's say 30000).
   * If they appeal and the round is already the 4th one(last one)
    * `GuardiansNumber` becomes the whole active balance divided by the `minActiveBalance` and the result multiplied by 1000 for division rounding.
    * we take `(_guardiansNumber * guardianFee) / FINAL_ROUND_WEIGHT_PRECISION) * finalRoundReduction / 10000 from the creator of the appeal.
 
 
* Then, comes `confirmAppeal`. If nobody confirms the above appeal and enough time gets passed, the dispute and round will be ended and the final ruling
will be whatever the `createAppeal` came up with. So, if someone makes an `appeal`, it's important that we confirm it, otherwise, `createdAppeal` just wins every time.
  * If the round that has to get created is the last one, we create the round and:
    * the dispute state becomes `Adjucating`, which means there'll be no `drafting` at all.
    * people now can vote on it, but since there was no draft and there was no choosen guardians, in this last round, anyone can vote if they have enough weight - `(activeBalance * FINAL_ROUND_WEIGHT_PRECISION / minActiveBalance) != 0 `. When they vote, we immediatelly take `activeBalance * penaltyPct / 10000` from their active balance.
  * If the round that has to get created is not the last one, we create the round and:
    * the dispute state becomes `PRE_DRAFT` , which means we can call `draft` and start choosing guardians as we were doing before. Everything above just gets repeated, but as we said above, `GuardiansNumber` that have to be chosen for the draft get `appealStepFactor times more than the current one`.
     


After everything IS DONE(final ruling should be calculated), we can start calling `settlePenalties` and `settleReward`. `settlePenalties` will punish the guardians that haven't voted in favor of the winner voice/rule. `settleReward` will reward those that voted in favor of the winner voice/rule.

The catch here is that `settlePenalties` have to be called separately for each round of the dispute. After calling it, that's when we can call `settleReward` for the same round.


`settlePenalties` :

* If the round isn't the last one, we take guardians' locked balance out of their account.
* If the round is the last one, we don't take anything, because we already took it from their account when they voted. This happens, because when the round is the last one, there's no draft. So, we can't lock any balance while drafting. That's why we immediatelly take it from their balance when they vote for the last round. If they want to get it back, they can call `settleReward`, but of course, they should have voted for the right answer.


-------

Graphs:

Courts:

* https://thegraph.com/explorer/subgraph/aragon/aragon-court-v2-staging (This is rinkeby staging)..
* https://thegraph.com/explorer/subgraph/aragon/aragon-court-v2-rinkeby 
* https://thegraph.com/explorer/subgraph/aragon/aragon-court-v2-mainnet

Govern:

* https://thegraph.com/explorer/subgraph/aragon/aragon-govern-rinkeby-staging (This is rinkeby staging)..
* https://thegraph.com/explorer/subgraph/aragon/aragon-govern-rinkeby
* https://thegraph.com/explorer/subgraph/aragon/aragon-govern-mainnet

------

Sites:

* court.aragon.org
* court-rinkeby.aragon.org

* v1.court.aragon.org
* v1.court.rinkeby.aragon.org

Quick References of function calls.


```js
await this.court.heartbeat(100);
await this.court.getDisputeFees();

await this.court.setConfig(
   657,
   "0xc7AD46e0b8a400Bb3C915120d284AafbA8fc4735", // this.config.feeToken.address
   [this.config.guardianFee, this.config.draftFee, this.config.settleFee],
   [this.config.evidenceTerms, this.config.commitTerms, this.config.revealTerms, this.config.appealTerms, this.config.appealConfirmTerms],
   [this.config.penaltyPct, this.config.finalRoundReduction],
   [this.config.firstRoundGuardiansNumber, this.config.appealStepFactor, this.config.maxRegularAppealRounds, this.config.finalRoundLockTerms],
   [this.config.appealCollateralFactor, this.config.appealConfirmCollateralFactor],
   this.config.minActiveBalance
)

await this.court.grant(
   soliditySha3(
     '0x6d2c871534B5De76a70333100533C579ddf57B2E',  // guardians registry
     // '0x2b7f012e' // stackeAndActivate
     '0x51d2f186' // lockActivation
   ), // id,
   "0x8A3475C25452B280a3Af1A8a9B4440e9f70f2f30" // anj lock minter,
)
```


### Rinkeby Network...

```
This is the first deployment of Protocol...... 2h term duration...

{
  "court": {
    "address": "0xC464EB732A1D2f5BbD705727576065C91B2E9f18",
    "transactionHash": "0x01d7995d4fcb7276b44d814819d7473c222e7e399863022c665e48b41faaa110"
  },
  "disputes": {
    "address": "0x6Afb995035057007BDc800D7e83B50b892eA4968",
    "transactionHash": "0x9a380dd42845e37fcd218cb348d18fe60c1d626a2e821c722918b8a67b65225d"
  },
  "registry": {
    "address": "0x6d2c871534B5De76a70333100533C579ddf57B2E",
    "transactionHash": "0x46af2515172532c5ca5dd025be6a29278445a5226f4cba5b53fa0c40674c714b"
  },
  "voting": {
    "address": "0xF01AC1bB2068998a48D3AFE1F3A26C1245b9D382",
    "transactionHash": "0xc4d1f7a2ae7cad87d3336863e986e403d69ea9783a18a3ee051ec6bb910409d2"
  },
  "treasury": {
    "address": "0x960ca7BecD6BF0232E8146591a480608f942e64F",
    "transactionHash": "0x53b12114c02f5835db23bcaa7d6f41ea5ff84c0d241ac86ceb90e9ab073045af"
  },
  "paymentsBook": {
    "address": "0x4d0161Badb05b6B80cD54CA917e195D5bE98bd5E",
    "transactionHash": "0x55d2dd59d58025937b9211d3d3fbe9a80ede5a337254b238e0655225d96c255f"
  }
}

This is on the rinkeby network too, but with term duration as 10 minutes.

{
  "court": {
    "address": "0xD2c15eCd1751C2cE8b02ab2D95db32E662517D61",
    "transactionHash": "0xc56cde607525f100037dbaf706098db04727e640d59f8659d492d16aff4d206a"
  },
  "disputes": {
    "address": "0xdc4db21C9Ba5226dCF3d0ce4D0e277AdE3AbCA40",
    "transactionHash": "0xce0cf8dfbc58c02f767c0f649a99433db0ecd4527a0dcfe2fbcd60d4dab6bf98"
  },
  "registry": {
    "address": "0x72f86668812DFBed665DD7879b3CF9CA05a51489",
    "transactionHash": "0xe207ed1abd92a7383c7484ae11cb66294792e2b631478bc27692130de448a652"
  },
  "voting": {
    "address": "0x3B0619a6bDaa145EebffD649852010C40Ff603a0",
    "transactionHash": "0xc347e89608f6d5b01d442ffd4006cd0c6ca2db1e1ace37ab023b7ac9003048aa"
  },
  "treasury": {
    "address": "0x831598961f14f477F5Ae3d23CeDc5b2b3c6eBb19",
    "transactionHash": "0x1c20477523aee52b546e0a5181d0ec88d6fd842028dc544a843c4452160098e4"
  },
  "paymentsBook": {
    "address": "0x932dCB8d618e150e3B0A5cCB1e62340bf39b7Cda",
    "transactionHash": "0x3f59cd6940e450b0462c671d812f7ad55551dc5d590564c6495310df5f7955d7"
  }
}

This is on rinkeby too but with 3 minutes term

{
  "court": {
    "address": "0x9c003eC97676c30a041f128D671b3Db2f790c3E7",
    "transactionHash": "0xae2b8fd1a35ee4162a2cb80f96c9a2450305c03099174f89645b65877137c397"
  },
  "disputes": {
    "address": "0xC0e572c9Ceb06AdFE2d754AFae16C9e2037D3B7f",
    "transactionHash": "0xd9454153ece2717422a8995320ed2b412e1f30fa9f8f4ef741d1ceabc9cb9c68"
  },
  "registry": {
    "address": "0xd96B7f14a3230A05C5D5c4Fa17a8C14ED3879d50",
    "transactionHash": "0x1a8dbf68221e39236e70185a7824a0b4680df5229b54094e12f1cec6274837ae"
  },
  "voting": {
    "address": "0x7b6470C4959d34b179B6120e49eA56Bf44fD5334",
    "transactionHash": "0xd94b62aa3a1e52c3a24e382c792d44dec63ff59b9f763f5740e2ae3607e64eba"
  },
  "treasury": {
    "address": "0xa59EeAC6640Fa44F28dFa5424e5e3a47a6ad76Dc",
    "transactionHash": "0x31f45bf8651b9f2f5b15e964da2e338d923ba13083f8d2fcafa5e77c6f04f72b"
  },
  "paymentsBook": {
    "address": "0x8D86Bf7FC6C58cc04391C8513Aa9A221eb5edE08",
    "transactionHash": "0x7e1dd75f19edd7d793ae35604c1663c3b66e77b6bde4bd2616be76b6f380d12b"
  }
}

```


### Mainnet

```
{
  "court": {
    "address": "0xFb072baA713B01cE944A0515c3e1e98170977dAF",
    "transactionHash": "0xb881141aadbc40f8b629ac0dd3cfda0f079a76a1cbbe3c0a52770165f5d7f0e1"
  },
  "disputes": {
    "address": "0x396677fEFeb2B321399847c82a2fC8D11985e746",
    "transactionHash": "0x31bd82ae043ec04ac06f484df51b0d5101bf0c178923983ec9cc59bd17f6eb72"
  },
  "registry": {
    "address": "0xab647b8fd9e370448d4eeb96582fe839f3d0bb24",
    "transactionHash": "0xb27beb8a76d7d55453f20a19e2f2eb990ab4290f092e54150426cc0d14815f86"
  },
  "voting": {
    "address": "0x81bc5c75aB0937cDBaD1F40ac585be6800a39448",
    "transactionHash": "0xe15e4c5b8fc0105c195365469ca748d65399dd344512b8a6ba6da0328fee83b5"
  },
  "treasury": {
    "address": "0x7f002a060e431aaB816058719d38dB9d94eF80bD",
    "transactionHash": "0x949f18d5d1b7f29e9824d679a9a2ab8fb46bc37eeec584094438b4cb5987a2fb"
  },
  "paymentsBook": {
    "address": "0x4D211A0fAF3572E37055d7Ef9e2631c7193d83A0",
    "transactionHash": "0x25c26a2b4f35de449fb551902cabfe57d47dbbd728c4d44bda399c463d09a004"
  }
}
```

