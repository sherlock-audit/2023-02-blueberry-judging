Jeiwan

medium

# Wrong amount of LP tokens is removed from ICHI vaults when closing a position

## Summary
When closing a position, users can remove wrapped LP tokens from the bank, unwrap them, and redeem them in ICHI vaults to repay their debt. The amount of redeemed LP tokens can be different from the amount of tokens removed from the bank, which can have different consequences:
1. if the amount of tokens to redeem equals the amount of tokens to remove from the bank (which is the standard scenario), the transaction will revert;
1. if the amount of tokens to redeem is smaller than the amount of tokens to remove from the bank, the remaining amount will be left in the spell contract (the contract, however, is not designed to hold LP tokens), and anyone will be able to use these tokens to repay their debt.
## Vulnerability Detail
The [closePosition](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L341) function and the [closePositionFarm](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L367) function are used to close a position, remove a collateral, remove LP tokens, and repay the debt of the position. The functions take two arguments to specify how many LP tokens to remove:
```solidity
* @param lpTakeAmt Amount of ICHI Vault LP token to take out from Bank
* @param amountLpWithdraw Amount of ICHI Vault LP to withdraw from ICHI Vault
```
However, the [actual amount](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L294-L295) of LP tokens to withdraw from ICHI Vault (the one that's expected to be set by `amountLpWithdraw`) is:
```solidity
// 2. Calculate actual amount to remove
uint256 amtLPToRemove = vault.balanceOf(address(this)) -
    amountLpWithdraw;

// 3. Withdraw liquidity from ICHI vault
vault.withdraw(amtLPToRemove, address(this));
```

I.e. it's the current amount of LP tokens the contract is holding (after withdrawing `lpTakeAmt` from the bank) minus `amountLpWithdraw`. This cause an unexpected behavior: users expect to withdraw `lpTakeAmt` LP tokens from the bank and redeem `amountLpWithdraw` LP tokens, but the actual redeemed amount is `lpTakeAmt - amountLpWithdraw` (assuming the contract isn't holding any LP tokens, which is true by design). This leads to the following situations:
1. If user wants to withdraw and redeem the same amount of LP tokens, their transaction may revert if the [maximal LTV check](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L325) fails: the redeemed amount of LP tokens will be 0, thus the repaid debt may be lower than expected.
1. If, due to the wrong calculation of LP tokens to redeem, some of the LP tokens are left in the contract, anyone else may use them to repay their debt by calling `closePosition`/`closePositionFarm` and setting `amountLpWithdraw` to 0: `amtLPToRemove` will equal the balance of the contract, which includes the leftover LPs of the previous user.
## Impact
1. Users may not be able to close positions and repay their debts by specifying `lpTakeAmt` and `amountLpWithdraw` as specified by the documentation of the `closePosition` and `closePositionFarm` functions.
1. Users may lose portions of their LP tokens when closing positions, letting other users to use these amounts to repay their debts.
## Code Snippet
[IchiVaultSpell.sol#L341](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L341)
[IchiVaultSpell.sol#L367](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L367)
[IchiVaultSpell.sol#L293-L295](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L293-L295)
## Tool used
Manual Review
## Recommendation
Consider updating the documentation of the `amountLpWithdraw` argument of the `closePosition` and `closePositionFarm` functions to let users know how actual amount of LP tokens to redeem from ICHI vaults is calculated. Or consider updating the `withdrawInternal` function to use the `amountLpWithdraw` argument as the actual amount of LP tokens to redeem or use the current contract's LP token balance as the amount (assuming it always equals to `lpTakeAmt`).