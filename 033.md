Bauer

high

# If a plugins is removed user will not able to get rewards  from that plugin

## Summary
If a plugins is removed user will not able to get rewards  from that plugin 

## Vulnerability Detail
The ```StakingModule``` contract allows user stakes some amount of TEL to earn potential rewards. User can call ```claim()``` to  claims yield from all plugins. However, if a plugin is removed user will not able to get rewards from that plugin 
```solidity
function _claimFromIndividualPlugin(address account, address to, address pluginAddress, bytes calldata auxData) private returns (uint256) {
        require(pluginsMapping[pluginAddress], "StakingModule::_claimFromIndividualPlugin: Provided pluginAddress is invalid");
        
        // balance of `to` before claiming
        uint256 balBefore = IERC20Upgradeable(tel).balanceOf(to);

        // xClaimed = "amount of TEL claimed from the plugin"
        uint256 xClaimed = IPlugin(pluginAddress).claim(account, to, parseAuxData(auxData)[pluginIndicies[pluginAddress]]);

        // we want to make sure the plugin did not return the wrong amount
        require(IERC20Upgradeable(tel).balanceOf(to) - balBefore == xClaimed, "The plugin did not send appropriate token amount");

        // only emit Claimed if anything was actually claimed
        if (xClaimed > 0) {
            emit Claimed(account, xClaimed);
        }

        return xClaimed;
    }
```

## Impact
 If a plugins is removed user will not able to get rewards

## Code Snippet
https://github.com/sherlock-audit/2023-02-telcoin/blob/main/telcoin-audit/contracts/staking/StakingModule.sol#L353-L377
https://github.com/sherlock-audit/2023-02-telcoin/blob/main/telcoin-audit/contracts/staking/StakingModule.sol#L543-L555
## Tool used

Manual Review

## Recommendation

