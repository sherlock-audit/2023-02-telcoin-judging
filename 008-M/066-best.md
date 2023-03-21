hyh

medium

# Adding and removing large set of addresses from blacklist may not be possible

## Summary

Currently balance is irreversibly lost for the user whenever addBlackList() is run, so removing a large number of accounts from blacklist can be not possible.

I.e. the returning balance transfer can be done manually, but only if old BLACKLISTER address remains available and honest and until there is big number of such accounts.

## Vulnerability Detail

If there be a need to remove a large amount of addresses from blacklist, with restoration of their balances, it might be not attainable as the balances are just transferred to BLACKLISTER previously and will require manual reconciliation for that.

Also, as balances were transferred to BLACKLISTER, whenever the current holder of this role be compromised (or, say, lost their private key), the balances are lost for good. I.e. it is possible to designate a new BLACKLISTER, but the balances are transferred to the address, not the role.

The reversal of blacklisting can be needed due to change of the protocol policies or as a fix of the operational mistake.

## Impact

Funds being expropriated from user accounts on blacklisting can be lost.

It might be impossible to remove a large set of users from blacklist with restoration of their balances due to technical limitations even if old BLACKLISTER address remained honest and functional.

## Code Snippet

addBlackList() besides setting the flag, calls removeBlackFunds():

https://github.com/sherlock-audit/2023-02-telcoin/blob/main/telcoin-audit/contracts/stablecoin/Stablecoin.sol#L152-L164

```solidity
  /**
   * @notice Adds an address to the mapping of blacklisted addresses
   * @dev See {IBlacklist-addBlackList}.
   * @param holder is the address being added to the blacklist
   *
   * Emits a {AddedBlacklist} event.
   */
  function addBlackList(address holder) external onlyRole(BLACKLISTER_ROLE) override {
    if (blacklisted(holder)) revert AlreadyBlacklisted(holder);
    _blacklist[holder] = true;
    emit AddedBlacklist(holder);
    removeBlackFunds(holder);
  }
```

Which transfer the funds to the current BLACKLISTER caller:

https://github.com/sherlock-audit/2023-02-telcoin/blob/main/telcoin-audit/contracts/stablecoin/Stablecoin.sol#L183-L186

```solidity
  function removeBlackFunds(address holder) internal {
    uint256 funds = balanceOf(holder);
    _transfer(holder, _msgSender(), funds);
  }
```

When address is cleared from blacklist, the funds attributed to them information is already lost and require manual reconciliation:

https://github.com/sherlock-audit/2023-02-telcoin/blob/main/telcoin-audit/contracts/stablecoin/Stablecoin.sol#L166-L177

```solidity
  /**
   * @notice Removes an address to the mapping of blacklisted addresses
   * @dev See {IBlacklist-removeBlackList}.
   * @param holder is the address being removed from the blacklist
   *
   * Emits a {RemovedBlacklist} event.
   */
  function removeBlackList(address holder) external onlyRole(BLACKLISTER_ROLE) override {
    if (!blacklisted(holder)) revert NotBlacklisted(holder);
    _blacklist[holder] = false;
    emit RemovedBlacklist(holder);
  }
```

## Tool used

Manual Review

## Recommendation

Reliance on availability and honesty of the current BLACKLISTER isn't advised in a long run.

Consider transferring balances of the blacklisted accounts to Stablecoin contract itself, keeping the record of such balances, and returning them back automatically in removeBlackList().

https://github.com/sherlock-audit/2023-02-telcoin/blob/main/telcoin-audit/contracts/stablecoin/Stablecoin.sol#L183-L186

```diff
  function removeBlackFunds(address holder) internal {
    uint256 funds = balanceOf(holder);
+   frozenFunds[holder] = funds;
-   _transfer(holder, _msgSender(), funds);
+   _transfer(holder, address(this), funds);
  }
```

https://github.com/sherlock-audit/2023-02-telcoin/blob/main/telcoin-audit/contracts/stablecoin/Stablecoin.sol#L28

```diff
  mapping (address => bool) private _blacklist;
+ mapping (address => uint256) private frozenFunds;
```

```diff
+   function returnBlackFunds(address holder) internal {
+     uint256 funds = frozenFunds[holder];
+     frozenFunds[holder] = 0;
+     _transfer(address(this), holder, funds);
+   }
```

https://github.com/sherlock-audit/2023-02-telcoin/blob/main/telcoin-audit/contracts/stablecoin/Stablecoin.sol#L166-L177

```diff
  /**
   * @notice Removes an address to the mapping of blacklisted addresses
   * @dev See {IBlacklist-removeBlackList}.
   * @param holder is the address being removed from the blacklist
   *
   * Emits a {RemovedBlacklist} event.
   */
  function removeBlackList(address holder) external onlyRole(BLACKLISTER_ROLE) override {
    if (!blacklisted(holder)) revert NotBlacklisted(holder);
    _blacklist[holder] = false;
+   returnBlackFunds(holder);
    emit RemovedBlacklist(holder);
  }
```