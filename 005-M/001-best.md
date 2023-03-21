volodya

medium

# `transferERCToBridge` will not work for some tokens that don't support approve `2**256 - 1` amount.

## Summary
`transferERCToBridge` will not work for some tokens that don't support approve `2**256 - 1` amount.
## Vulnerability Detail

## Impact
There are tokens that don't support approve spender `2**256 - 1` amount. So the transferERCToBridge will not work for some tokens like UNI who will revert when approve `2**256 - 1` amount. `Uni` is on the list that project promise to support

```solidity
    function approve(address spender, uint rawAmount) external returns (bool) {
        uint96 amount;
        if (rawAmount == uint(-1)) {
            amount = uint96(-1);
        } else {
345:       amount = safe96(rawAmount, "Uni::approve: amount exceeds 96 bits");
        }

        allowances[msg.sender][spender] = amount;

        emit Approval(msg.sender, spender, amount);
        return true;
    }
```
[code#L345](https://etherscan.io/token/0x1f9840a85d5af5bf1d1762f925bdaddc4201f984#code#L345)
## Code Snippet
```solidity
31:  uint256 constant public MAX_INT = 2**256 - 1;

70:  if (balance > IERC20Upgradeable(token).allowance(recipient, PREDICATE_ADDRESS)) {IERC20Upgradeable(token).safeApprove(PREDICATE_ADDRESS, MAX_INT);}

```
[RootBridgeRelay.sol#L70](https://github.com/sherlock-audit/2023-02-telcoin/blob/main/telcoin-audit/contracts/bridge/RootBridgeRelay.sol#L70)
## Tool used

Manual Review

## Recommendation
```solidity
if (balance > IERC20Upgradeable(token).allowance(recipient, PREDICATE_ADDRESS)) {
IERC20Upgradeable(token).safeApprove(PREDICATE_ADDRESS,
      balance - IERC20Upgradeable(token).allowance(recipient, PREDICATE_ADDRESS) 
    );}
    
}

```