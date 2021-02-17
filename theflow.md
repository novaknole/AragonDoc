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