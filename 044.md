SeWizarD

medium

# Issue 1: Lack of recovery mechanism for stuck Ether

## Summary:
The contract does not provide a mechanism to recover Ether that may get stuck in the contract due to unforeseen circumstances. This can lead to a permanent loss of Ether for the users.

## Vulnerability Detail:
The contract includes an `erc20Rescue` function to recover MATIC tokens but lacks a similar function for Ether recovery. In case Ether gets stuck in the contract, there is no way to retrieve the stuck funds.

## Impact:
Inability to recover stuck Ether may result in a permanent loss of funds and undermine user trust in the contract.

## Code Snippet
https://github.com/sherlock-audit/2023-02-telcoin/blob/main/telcoin-audit/contracts/bridge/RootBridgeRelay.sol#L80-L83

## Tool used
Manual Review

## Recommendation:
Implement a function to recover Ether stuck in the contract.

``` solidity
function ethRescue(address payable destination, uint256 amount) external{
    require(msg.sender == _owner, "RootBridgeRelay: caller must be owner");
    destination.transfer(amount);
} ```

