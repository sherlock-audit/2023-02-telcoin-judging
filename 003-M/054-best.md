Tricko

medium

# `slash` calls can be blocked, allowing malicious users to bypass the slashing mechanism.

## Summary
A malicious user can block slashing by frontrunning `slash` with a call to `stake(1)` at the same block, allowing him to keep blocking calls to `slash` while waiting for his withdraw delay, effectively bypassing the slashing mechanism.

## Vulnerability Detail
`StakingModule`'s `checkpointProtection` modifier reverts certain actions, like claims, if the accounts' stake was previously modified in the same block. A malicious user can exploit this to intentionally block calls to `slash`.

Consider the following scenario, where Alice has `SLASHER_ROLE` and Bob is the malicious user.
1. Alice calls `slash` on Bob's account.
2. Bob sees the transaction on the mempool and tries to frontrun it by staking 1 TEL.
(See Proof of Concept section below for a simplified example of this scenario)

If Bob stake call is processed first (he can pay more gas to increase his odds of being placed before than Alice), his new stake is pushed to `_stakes[address(Bob)]`, and his latest checkpoint (`_stakes[address(Bob)]._checkpoints[numCheckpoints - 1]`) `blockNumber` field is updated to the current `block.number`. So when `slash` is being processed in the same block and calls internally `_claimAndExit` it will revert due to the `checkpointProtection` modifier check (See code snippet below).

```javascript
modifier checkpointProtection(address account) {
    uint256 numCheckpoints = _stakes[account]._checkpoints.length;
    require(numCheckpoints == 0 || _stakes[account]._checkpoints[numCheckpoints - 1]._blockNumber != block.number, "StakingModule: Cannot exit in the same block as another stake or exit");
    _;
}
```
Bob can do this indefinitely, eventually becoming a gas war between Alice and Bob or until Alice tries to use Flashbots Protect or similar services to avoid the public mempool. More importantly, this can be leverage to block all `slash` attempts while waiting the time required to withdraw, so the malicious user could call `requestWithdrawal()`, then keep blocking all future `slash` calls while waiting for his `withdrawalDelay`, then proceed to withdraws his stake when `block.timestamp > withdrawalRequestTimestamps[msg.sender] + withdrawalDelay`. Therefore bypassing the slashing mechanism.

In this modified scenario 
1. Alice calls `slash` on Bob's account.
2. Bob sees the transaction on the mempool and tries to frontrun it by staking 1 TEL.
3. Bob requests his withdraw (`requestWithdrawal()`)
4. Bob keeps monitoring the mempool for future calls to `slash` against his account, trying to frontrun each one of them.
5. When enough time has passed so that his withdraw is available, Bob calls `exit` or `fullClaimAndExit`

## Impact
Slashing calls can be blocked by malicious user, allowing him to request his withdraw, wait until withdraw delay has passed (while blocking further calls to `slash`) and then withdraw his funds.

Classify this one as medium severity, because even though there are ways to avoid being frontrunned, like paying much more gas or using services like Flashbots Protect, none is certain to work because the malicious user can use the same methods to their advantage.  And if the malicious user is successful, this would result in loss of funds to the protocol (i.e funds that should have been slashed, but user managed to withdraw them)

## Proof of Concept
The POC below shows that staking prevents any future call to `slash` on the same block. To reproduce this POC just copy the code to a file on the test/ folder and run it.
```javascript
const { expect } = require("chai")
const { ethers, upgrades } = require("hardhat")

const emptyBytes = []

describe("POC", () => {
  let deployer
  let alice
  let bob
  let telContract
  let stakingContract
  let SLASHER_ROLE

  beforeEach("setup", async () => {
    [deployer, alice, bob] = await ethers.getSigners()

    //Deployments
    const TELFactory = await ethers.getContractFactory("TestTelcoin", deployer)
    const StakingModuleFactory = await ethers.getContractFactory(
      "StakingModule",
      deployer
    )
    telContract = await TELFactory.deploy(deployer.address)
    await telContract.deployed()
    stakingContract = await upgrades.deployProxy(StakingModuleFactory, [
      telContract.address,
      3600,
      10
    ])

    //Grant SLASHER_ROLE to Alice
    SLASHER_ROLE = await stakingContract.SLASHER_ROLE()
    await stakingContract
      .connect(deployer)
      .grantRole(SLASHER_ROLE, alice.address)

    //Send some TEL tokens to Bob
    await telContract.connect(deployer).transfer(bob.address, 1)

    //Setup approvals
    await telContract
      .connect(bob)
      .approve(stakingContract.address, 1)
  })

  describe("POC", () => {
    it("should revert during slash", async () => {
      //Disable auto-mining and set interval to 0 necessary to guarantee both transactions
      //below are mined in the same block, reproducing the frontrunning scenario.
      await network.provider.send("evm_setAutomine", [false]);
      await network.provider.send("evm_setIntervalMining", [0]);

      //Bob stakes 1 TEL
      await stakingContract
        .connect(bob)
        .stake(1)

      //Turn on the auto-mining, so that after the next transaction is sent, the block is mined.
      await network.provider.send("evm_setAutomine", [true]);
      
      //Alice tries to slash Bob, but reverts.
      await expect(stakingContract
        .connect(alice)
        .slash(bob.address, 1, stakingContract.address, emptyBytes)).to.be.revertedWith(
          "StakingModule: Cannot exit in the same block as another stake or exit"
        )
    })
  })
})
```

## Code Snippet

https://github.com/sherlock-audit/2023-02-telcoin/blob/main/telcoin-audit/contracts/staking/StakingModule.sol#L109-L113

https://github.com/sherlock-audit/2023-02-telcoin/blob/main/telcoin-audit/contracts/staking/StakingModule.sol#L510-L513

https://github.com/sherlock-audit/2023-02-telcoin/blob/main/telcoin-audit/contracts/staking/StakingModule.sol#L460-L483

## Tool used
Manual Review

## Recommendation
Consider implementing a specific version of `_claimAndExit` without the `checkpointProtection` modifier, to be used inside the `slash` function. 
