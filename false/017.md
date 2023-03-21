jonatascm

medium

# Malfunction in `transferERCToBridge` will eventually lead to stop bridging tokens

## Summary

The `transferERCToBridge` uses the `safeApprove` function which checks the value of the token allowance, in case there is already an allowance it will revert DOSing the token transfers.

## Vulnerability Detail

The `transferERCToBridge` function in the `RootBridgeRelay` contract uses the `safeApprove` function, setting the value to MAX_INT, it will eventually need to add more allowance, but if the token allowance is different than 0 the `safeApprove` function will revert, disabling the bridge to this token.

## POC

Using the `RootBridgeRelay.js` test we could make this POC

```solidity
it.only("transferERCToBridgeDOS", async () => {
  //@audit this help to change the allowance of bridge
  await deployer.sendTransaction({ to: rootBridgeRelay.address, value: ethers.utils.parseEther("1.0") });
  await hre.network.provider.request({
      method: "hardhat_impersonateAccount",
      params: [rootBridgeRelay.address],
  });
  const bridge = await ethers.getSigner(rootBridgeRelay.address);

  //@audit make the token allowance more than 0 and different from transferred value: 0.5 ether
  await telcoin.connect(bridge).approve(predicate.address, ethers.utils.parseEther("0.5"));

  //@audit end impersonating
  await hre.network.provider.request({
      method: "hardhat_stopImpersonatingAccount",
      params: [rootBridgeRelay.address],
  });

  
  await telcoin.connect(deployer).transfer(rootBridgeRelay.address, ethers.utils.parseEther("1.0"))
  expect(await telcoin.balanceOf(rootBridgeRelay.address)).to.equal(ethers.utils.parseEther("1.0"));
	//@audit this call will fail because of safeApprove function
  await expect(rootBridgeRelay.connect(deployer).bridgeTransfer(telcoin.address)).to.be.reverted;
});
```

## Impact

The users couldn't bridge the token that entered in this state.

## Code Snippet

https://github.com/sherlock-audit/2023-02-telcoin/blob/main/telcoin-audit/contracts/bridge/RootBridgeRelay.sol#L67-L72

```solidity
function transferERCToBridge(address token) internal {
  uint256 balance = IERC20Upgradeable(token).balanceOf(address(this));
  if (balance > IERC20Upgradeable(token).allowance(recipient, PREDICATE_ADDRESS)) {IERC20Upgradeable(token).safeApprove(PREDICATE_ADDRESS, MAX_INT);}
  POS_BRIDGE.depositFor(recipient, token, abi.encodePacked(balance));
  emit BridgeERC(recipient, token, balance);
}
```

## Tool used

Manual Review

## Recommendation

It's recommended to use safeApprove to set it to 0 before set again to MAX_INT or only approve the amount to be transferred:

```diff
function transferERCToBridge(address token) internal {
  uint256 balance = IERC20Upgradeable(token).balanceOf(address(this));
  if (balance > IERC20Upgradeable(token).allowance(recipient, PREDICATE_ADDRESS)) {
+		IERC20Upgradeable(token).safeApprove(PREDICATE_ADDRESS, 0);
		IERC20Upgradeable(token).safeApprove(PREDICATE_ADDRESS, MAX_INT);
	}
  POS_BRIDGE.depositFor(recipient, token, abi.encodePacked(balance));
  emit BridgeERC(recipient, token, balance);
}
```