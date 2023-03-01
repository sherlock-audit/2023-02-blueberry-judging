sinarette

medium

# Failing to call depositInternal / withdrawInternal with zero input

## Summary
````IchiVaultSpell#depositInternal```` calls the ````ICHIVault#deposit```` function, and ````IchiVaultSpell#withdrawInternal```` calls the ````ICHIVault#withdraw```` function from a given ICHI vault. There may be occasions that the deposit/withdraw amount parameter ````balance/amtLPToRemove```` is zero, but the ````deposit()/withdraw()```` function would revert on zero input. As a result, users would be unable to conduct their intended actions.

## Vulnerability Detail
```solidity
    function depositInternal(
    ...
        // 2. Borrow specific amounts
        doBorrow(borrowToken, borrowAmount);

        // 3. Add liquidity - Deposit on ICHI Vault
        IICHIVault vault = IICHIVault(strategy.vault);
        bool isTokenA = vault.token0() == borrowToken;
        uint256 balance = IERC20(borrowToken).balanceOf(address(this));
        ensureApprove(borrowToken, address(vault));
        if (isTokenA) {
            vault.deposit(balance, 0, address(this)); //@audit revert if balance == 0
        } else {
            vault.deposit(0, balance, address(this));
        }
```

````depositInternal```` borrows some ````borrowToken```` from the bank contract, then deposits those tokens by calling ````vault.deposit````. If ````borrowAmount``` was zero, then it would simply ignore and pass the lending action, so that the remaining actions could run normally.
```solidity
    function doBorrow(address token, uint256 amount) internal {
        if (amount > 0) {   //@audit No Revert
            bank.borrow(token, amount);
        }
    }
```

As the contract borrowed no tokens and no extra tokens should be kept in the contract, ````balance```` would be zero too. However, ````vault.deposit```` would revert on zero input.
In ICHIVault:
```solidity
2572:    function deposit(

2580:        require(
2581:            deposit0 > 0 || deposit1 > 0,
2582:            "IV.deposit: deposits must be > 0"
2583:        );
```

Similarly ````withdrawInternal```` would revert in case that ````amountLpWithdraw```` is equal to ````lpTakeAmt````.

```solidity
        // 2. Calculate actual amount to remove
        uint256 amtLPToRemove = vault.balanceOf(address(this)) -
            amountLpWithdraw;

        // 3. Withdraw liquidity from ICHI vault
        vault.withdraw(amtLPToRemove, address(this));
```

````withdrawInternal()```` is called from ````closePosition()```` and ````closePositionFarm()````, after taking out the LP tokens by calling ````doTakeCollateral()````.
```solidity
    function closePosition(
    ...
        // 1. Take out collateral
        doTakeCollateral(strategies[strategyId].vault, lpTakeAmt);

        // 2-8. Remove liquidity
        withdrawInternal(
            strategyId,
            collToken,
            borrowToken,
            amountRepay,
            amountLpWithdraw,
            amountShareWithdraw
        );

    function closePositionFarm(
    ...
        // 1. Take out collateral
        bank.takeCollateral(lpTakeAmt);
        wIchiFarm.burn(collId, lpTakeAmt);
        doCutRewardsFee(ICHI);

        // 2-8. Remove liquidity
        withdrawInternal(
            strategyId,
            collToken,
            borrowToken,
            amountRepay,
            amountLpWithdraw,
            amountShareWithdraw
        );
```

If the user wants to keep all the withdrawal as LP tokens, the user can simply input ````amountLpWithdraw```` to be equal to ````lpTakeAmt````, as the ````vault.balanceOf(address(this))```` should be same as ````lpTakeAmt````.
However, if then, ````amtLPToRemove```` would be zero, which leads to revert in ````vault.withdraw````.

in ICHIVault:
```solidity
2669:    function withdraw(uint256 shares, address to)
2674:    {
2675:        require(shares > 0, "IV.withdraw: shares");
```

## Impact
Users would fail to open/adjust position without borrowing
Users are unable to keep their withdrawal as LP tokens

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L122-L157
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L276-L330
https://etherscan.io/address/0x683F081DBC729dbD34AbaC708Fa0B390d49F1c39#code

## Tool used

Manual Review

## Recommendation
Add checking logic before deposit/withdraw
```solidity
        uint256 balance = IERC20(borrowToken).balanceOf(address(this));
+       if(balance > 0) {
            ensureApprove(borrowToken, address(vault));
            if (isTokenA) {
                vault.deposit(balance, 0, address(this)); //@audit revert if balance == 0
            } else {
                vault.deposit(0, balance, address(this));
            }
+       }
```
```solidity
        // 3. Withdraw liquidity from ICHI vault
+       if(amtLPToRemove > 0)
            vault.withdraw(amtLPToRemove, address(this));
```


