hyh

medium

# Account that is affiliated with a plugin can sometimes evade slashing

## Summary

Rogue plugin can be a big staker itself or can collide with one and allow such staker to evade slashing in a number of scenarios, i.e. reduce the probability of slashing execution.

## Vulnerability Detail

In order to achieve that the `plugin` can behave otherwise normally in all regards, but on observing staked amount reduction for a specific `account` it can revert `IPlugin(plugin).requiresNotification()`.

If a given `account` also partially mitigate the existence of `withdrawalDelay > 0` with the periodic renewal of withdrawal requests (without using any, just to have some window available), the overall probability of it to be able to withdraw while `plugin` is still in the system is noticeable.

This way the overall scenario is:

1. `plugin` and `account` collide and set up the monitoring
2. SLASHER's slash() for `account` is front-run with `plugin` tx switching its state so it is now reverting on `IPlugin(plugin).requiresNotification()`
3. slash() is reverted this way, `plugin` switches to a normal state (it basically sandwiches slashing with two txs, own state change forth and back)
4. SLASHER investigate with PLUGIN_EDITOR who the reverting plugin is
5. Meanwhile withdraw window `account` has requested beforehand is approaching and if it occurs before PLUGIN_EDITOR removes a plugin (the ability to do so is an another issue, here we suppose it's fixed and plugin is removable) the `account` will be able to withdraw fully
6. `account` exit() executes as `plugin` is in normal state and doesn't block anything

## Impact

`account` have some chance to evade the slashing, withdrawing the whole stake before slashing can occur.

With the growth of the protocol and increasing of the number of plugins this probability will gradually raise as volatile behavior of a particular plugin can be more tricky to identify which can provide enough time for an `account`.

The cost of being removed can be bearable for `plugin` provided that the `account` stake saved is big enough.

## Code Snippet

Plugin can revert the `IPlugin(plugin).requiresNotification()` call:

https://github.com/sherlock-audit/2023-02-telcoin/blob/main/telcoin-audit/contracts/staking/StakingModule.sol#L485-L498

```solidity
    /// @dev Calls `notifyStakeChange` on all plugins that require it. This is done in case any given plugin needs to do some stuff when a user exits.
    /// @param account Account that is exiting
    function _notifyStakeChangeAllPlugins(address account, uint256 amountBefore, uint256 amountAfter) private {
        // loop over all plugins
        for (uint256 i = 0; i < nPlugins; i++) {
            // only notify if the plugin requires
>>          if (IPlugin(plugins[i]).requiresNotification()) {
                try IPlugin(plugins[i]).notifyStakeChange(account, amountBefore, amountAfter) {}
                catch {
                    emit StakeChangeNotificationFailed(plugins[i]);
                }
            }
        }
    }
```

It will prohibit slashing as slash() calls _claimAndExit() that invokes _notifyStakeChangeAllPlugins():

https://github.com/sherlock-audit/2023-02-telcoin/blob/main/telcoin-audit/contracts/staking/StakingModule.sol#L510-L513

```solidity
    function slash(address account, uint amount, address to, bytes calldata auxData) external onlyRole(SLASHER_ROLE) nonReentrant {
        _claimAndExit(account, amount, to, auxData);
        emit Slashed(account, amount);
    }
```

https://github.com/sherlock-audit/2023-02-telcoin/blob/main/telcoin-audit/contracts/staking/StakingModule.sol#L460-L471

```solidity
    function _claimAndExit(address account, uint256 amount, address to, bytes calldata auxData) private checkpointProtection(account) {
        require(amount <= balanceOf(account, auxData), "Account has insufficient balance");

        // keep track of initial stake
        uint256 oldStake = _stakes[account].latest();
        // xClaimed = total amount claimed
        uint256 xClaimed = _claim(account, address(this), auxData);

        uint256 newStake = oldStake + xClaimed - amount;

        // notify all plugins that account's stake has changed (if the plugin requires)
>>      _notifyStakeChangeAllPlugins(account, oldStake, newStake);
```

If there is a `withdrawalDelay` the `account` can routinely renew withdrawal requests:

https://github.com/sherlock-audit/2023-02-telcoin/blob/main/telcoin-audit/contracts/staking/StakingModule.sol#L231-L236

```solidity
    function requestWithdrawal() external {
        require(withdrawalDelay > 0, "StakingModule: Withdrawal delay is 0");
        require(block.timestamp > withdrawalRequestTimestamps[msg.sender] + withdrawalDelay + withdrawalWindow, "StakingModule: Withdrawal already pending");

        withdrawalRequestTimestamps[msg.sender] = block.timestamp;
    }
```

This way there is a chance that `account` will be able to withdraw while SLASHER locates the reason of blocking and communicate with PLUGIN_EDITOR in order to remove the `plugin`.

## Tool used

Manual Review

## Recommendation

Consider adding `try-catch` to the requiresNotification() call, for example:

https://github.com/sherlock-audit/2023-02-telcoin/blob/main/telcoin-audit/contracts/staking/StakingModule.sol#L485-L498

```diff
    /// @dev Calls `notifyStakeChange` on all plugins that require it. This is done in case any given plugin needs to do some stuff when a user exits.
    /// @param account Account that is exiting
    function _notifyStakeChangeAllPlugins(address account, uint256 amountBefore, uint256 amountAfter) private {
        // loop over all plugins
        for (uint256 i = 0; i < nPlugins; i++) {
            // only notify if the plugin requires
-           if (IPlugin(plugins[i]).requiresNotification()) {
+           bool notificationRequired;
+           try IPlugin(plugins[i]).requiresNotification() returns (bool req) { notificationRequired = req; }
+           catch  { emit StakeChangeNotificationFailed(plugins[i]); }
+           if (notificationRequired) {
                try IPlugin(plugins[i]).notifyStakeChange(account, amountBefore, amountAfter) {}
                catch {
                    emit StakeChangeNotificationFailed(plugins[i]);
                }
            }
        }
    }
```