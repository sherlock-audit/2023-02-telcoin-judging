csanuragjain

medium

# Claim is done on deactivated plugin

## Summary
It seems that it is not checked whether a plugin is disabled before making a claim.

## Vulnerability Detail
1. Assume that plugin X was deactivated
2. Owner has still not removed the plugin
3. User immediately makes a claim using `claim` function which internally calls `_claim`
4. Now `_claim` does not check whether plugin is deactivated which means claim will be called over a deactivated plugin which was not expected

```solidity
function _claim(address account, address to, bytes calldata auxData) private returns (uint256) {
        // balance of `to` before claiming
        uint256 balBefore = IERC20Upgradeable(tel).balanceOf(to);

        // call claim on all plugins and count the total amount claimed
        uint256 total;
        bytes[] memory parsedAuxData = parseAuxData(auxData);
        for (uint256 i = 0; i < nPlugins; i++) {
            try IPlugin(plugins[i]).claim(account, to, parsedAuxData[i]) returns (uint256 xClaimed) {
                total += xClaimed;
            } catch  {
                emit PluginClaimFailed(plugins[i]);
            }
        }
...
}
```

## Impact
User can make claim from a disabled plugin which is not expected

## Code Snippet
https://github.com/sherlock-audit/2023-02-telcoin/blob/main/telcoin-audit/contracts/staking/StakingModule.sol#L281

## Tool used
Manual Review

## Recommendation
Before claiming amount in a plugin, it should be checked that plugin is not disabled

```solidity
function _claim(address account, address to, bytes calldata auxData) private returns (uint256) {
        // balance of `to` before claiming
        uint256 balBefore = IERC20Upgradeable(tel).balanceOf(to);

        // call claim on all plugins and count the total amount claimed
        uint256 total;
        bytes[] memory parsedAuxData = parseAuxData(auxData);
        for (uint256 i = 0; i < nPlugins; i++) {

if(IPlugin(plugin).deactivated()){
continue;
}

            try IPlugin(plugins[i]).claim(account, to, parsedAuxData[i]) returns (uint256 xClaimed) {
                total += xClaimed;
            } catch  {
                emit PluginClaimFailed(plugins[i]);
            }
        }
...
}
```