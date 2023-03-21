J4de

medium

# `addBlackList` function can be frontrunned to transfer assets in advance

## Summary

`addBlackList` function can be frontrunned to transfer assets in advance.

## Vulnerability Detail

Attackers can listen to the mempool and frontrun the `addBlackList` function call by BLACKLISTER_ROLE to transfer assets in advance before to be `removeBlackFunds`.

## Impact

Attackers can bypass the blacklist mechanism.

## Code Snippet

https://github.com/sherlock-audit/2023-02-telcoin/blob/main/telcoin-audit/contracts/stablecoin/Stablecoin.sol#L159-L164

## Tool used

Manual Review

## Recommendation
