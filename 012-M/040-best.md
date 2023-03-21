banditx0x

high

# Multiple roles can directly steal user funds

## Summary

Note: I submitted this as high, as these are roles that are not supposed to be able to rug people's funds. They violate the principle of least privilege.

## Vulnerability Detail

There are 3 roles in the contract that can be used to transfer funds to themselves:

`SLASHER_ROLE`
`BURNER_ROLE`
`BLACKLISTER_ROLE`

Although protocol rugs are not usually considered as a high vulnerability, this case is unique as there are 3 seperate roles that all can decrease a users token balance without their balance even being staked in the contract. In addition, there is a direct profit incentive for these role-holders to sabotage other users funds - the funds are transferred directly into their own wallets.

For example:

When `BLACKLISTER_ROLE` blacklists an address, all of the funds are transfered to them:   

function removeBlackFunds(address holder) internal {
    uint256 funds = balanceOf(holder);
    _transfer(holder, _msgSender(), funds);
  }


## Impact

There are 3 separate roles/addresses which can steal user funds for profit, which means that users are required to trust 3 separate roles not to steal their tokens, even if their tokens are not staked.

## Code Snippet

https://github.com/sherlock-audit/2023-02-telcoin/blob/main/telcoin-audit/contracts/stablecoin/Stablecoin.sol#L183-L186

https://github.com/sherlock-audit/2023-02-telcoin/blob/main/telcoin-audit/contracts/staking/StakingModule.sol#L510

## Tool used

Manual Review

## Recommendation

Remove these roles altogether, change these to multisigs or reduce their level of privilege. 
