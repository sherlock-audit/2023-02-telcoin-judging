spyrosonic10

medium

# FeeBuyback.submit() method may fail if all allowance is not used by referral contract

## Summary
Inside `submit()` method of `FeeBuyback.sol`, if token is `_telcoin` then it safeApprove to `_referral` contract.   If `_referral` contract do not use all allowance then `submit()` method will fail in next call. 

## Vulnerability Detail
`SafeApprove()` method of library `SafeERC20Upgradeable` revert in following scenario. 
```solidity
require((value == 0) || (token.allowance(address(this), spender) == 0), 
"SafeERC20: approve from non-zero to non-zero allowance");
```
Submit method is doing `safeApproval` of Telcoin to referral contract.  If referral contract do not use full allowance then subsequent call to submit() method will fails because of `SafeERC20: approve from non-zero to non-zero allowance`.  `FeeBuyback` contract should not trust or assume that referral contract will use all allowance.  If it does not use all allowance in `increaseClaimableBy()` method then submit() method will revert in next call. This vulnerability exists at two places in `submit()` method.  Link given in code snippet section.

## Impact
Submit() call will fail until referral contract do not use all allowance.

## Code Snippet
https://github.com/sherlock-audit/2023-02-telcoin/blob/main/telcoin-audit/contracts/staking/FeeBuyback.sol#L63-L64

https://github.com/sherlock-audit/2023-02-telcoin/blob/main/telcoin-audit/contracts/staking/FeeBuyback.sol#L63-L64

## Tool used

Manual Review

## Recommendation
Reset allowance to 0 before non-zero approval.

```solidity
_telcoin.safeApprove(address(_referral), 0);
_telcoin.safeApprove(address(_referral), _telcoin.balanceOf(address(this)));
```
