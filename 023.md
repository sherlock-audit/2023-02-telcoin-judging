spyrosonic10

high

# Withdraw delay can be bypassed

## Summary
StakingModule has core feature around staking, claim and withdraw. All these features has core and essential mechanism which is `delayed withdrawal`. In ideal scenario, user will stake X amount of token and will call `requestWithdrawal` when user want to withdraw his/her stake. `requestWithdrawal` will record user's request to withdraw and allow this user to withdraw only after `withdrawalDelay` is passed and during `withdrawalWindow` only. User can call `requestWithdrawal` in well advance before staking and this will allow user to bypass `withdrawalDelay`.

## Vulnerability Detail
Withdraw locking/delaying is core feature of this contract and it can be exploited very easily.

User can call `requestWithdrawal` before staking tokens and this will set user's `withdrawalRequestTimestamps`. Once `withdrawalDelay` is passed user can easily stake and unstake without locking time.

One would suggest that easy fix is to check `staked > 0` during call to `requestWithdrawal` and that should solve this issue. No, it will not.
Assume `staked>0` check is added in `requestWithdrawal` then user will stake 1 wei and call `requestWithdrawal` and this will result in almost same scenario. 

Why would this happen?
Because there is no relationship between stake and `withdrawalRequestTimestamps`.

## Impact
`withdrawalDelay` can be bypassed

## Code Snippet
https://github.com/sherlock-audit/2023-02-telcoin/blob/main/telcoin-audit/contracts/staking/StakingModule.sol#L231-L236

https://github.com/sherlock-audit/2023-02-telcoin/blob/main/telcoin-audit/contracts/staking/StakingModule.sol#L240-L246

**POC**
```js
      it.only("should bypass withdrawal delay", async () => {
        const delay = 60
        const window = 30
        // Set time delay
        await stakingContract.connect(deployer).grantRole(SLASHER_ROLE, slasher.address)
        await stakingContract.connect(slasher).setWithdrawDelayAndWindow(delay, window)
        await helpers.mine(1)
        // Request withdrawal
        await stakingContract.requestWithdrawal()
        // Increase time to pass timed delay
        await helpers.time.increase(delay)
        // Stake some tokens
        bobStakeTx3 = await stakingContract.connect(bob).stake(bobAmtStake)
        // Check there is non-zero staked balance
        expect(await stakingContract.balanceOf(bob.address, emptyBytes)).gt(0)
        // Claim and exit without wait.
        await stakingContract.fullClaimAndExit(emptyBytes)
      })
```

## Tool used

Manual Review

## Recommendation
Consider resetting `withdrawalRequestTimestamps` when user stake any amount of token.

```solidity
    function stake(uint256 amount) external nonReentrant {
        _stake({
            account: msg.sender, 
            from: msg.sender, 
            amount: amount
        });
        withdrawalRequestTimestamps = 0;
    }
```