chaduke

medium

# beforeTokenTransfer() fails to prevent the blacklisted owner from transferring the funds to another account

## Summary
``beforeTokenTransfer()`` fails to prevent the blacklisted owner from transferring the funds to another account. 

A blacklisted owner should not be allowed to transfer funds in-and-out of the blacklisted address. The freeze should in both ways. The only exception is BLACKLISTER_ROLE, who can transfer the funds out of the blacked listed account to another account. 

## Vulnerability Detail
``beforeTokenTransfer()`` prevents a user from sending funds to a blacklisted address; however, it does not prevent the owner from transfering existing funds away from the blacklisted account. 

[https://github.com/sherlock-audit/2023-02-telcoin/blob/main/telcoin-audit/contracts/stablecoin/Stablecoin.sol#L192-L194]https://github.com/sherlock-audit/2023-02-telcoin/blob/main/telcoin-audit/contracts/stablecoin/Stablecoin.sol#L192-L194)

This compromised the purpose of the blacklisting: maybe the funds is illegal and one needs to snatch/freeze the illegal funds. So it is important to freeze the transfer in both directions. The only exception is to allow a BLACKLISTER_ROLE user to transfer the funds from the blacklisted address to anther address. 

## Impact
Impact: once a user realizes his account is blacklisted, he can quickly transfer the funds to another account, as a result, he can transfer the possibly illegal funds to somewhere else. The functionality of blacklisting fails. 


## Code Snippet
See above

## Tool used
VSCode

Manual Review

## Recommendation
We will freeze transfer in both day, only giving exception to BLACKLISTER_ROLE.
```diff
- function _beforeTokenTransfer(address, address destination, uint256) internal view
+ function _beforeTokenTransfer(address from, address destination, uint256) internal  view
                  override(ERC20Upgradeable, ERC20PresetMinterPauserUpgradeable) {

    require(!blacklisted(destination), "Stablecoin: destination cannot be blacklisted address");
+    if(blacklisted(from)) require(hasRole(BLACKLISTER_ROLE, _msgSender()), "Must have BLACKLISTER_ROLE to transfer out from this address.");
  }

```