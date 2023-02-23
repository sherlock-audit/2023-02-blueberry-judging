Jeiwan

high

# Full debt of a position cannot be repaid, forcing users into unavoidable liquidations

## Summary
In `IchiVaultSpell`, positions can be closed and repaid only by redeeming LP tokens and selling underlying vault tokens for the borrowed tokens. However, by design, the value of debt can be greater than the value of LP tokens. Thus, the entire balance of LP tokens of a position may be not enough to repay the entire debt of the position.
## Vulnerability Detail
The [risk of a position is calculated as](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L493):
```solidity
if (cv == 0) risk = 0;
else if (pv >= ov) risk = 0;
else {
    risk = ((ov - pv) * DENOMINATOR) / cv;
}
```

Where, `ov` is the position's debt, `pv` is the value of the LP tokens of the position, `cv` is the value of the isolated collateral of the position. I.e. it's the ratio of a debt value to a collateral value, where debt is calculated as the borrowed amount (including accrued interest) minus the value of the LP tokens (which are redeemable for the tokens that were borrowed).

For a risk to be non-zero, the value of debt (`ov`) must be greater than the value of LP tokens (`pv`). The protocol is designed for users to take debts, for the debts to accrue interest and be repaid. Thus, it's by design that `ov` will be greater than `pv`.

However, when closing positions and repaying debts via `IchiVaultSpell`, debt is repaid with LP tokens:
1. first, [LP tokens are removed from the bank](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L355);
1. then, the removed LP tokens are [redeemed for the underlying tokens in the ICHI vault](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L298);
1. one of the redeemed underlying tokens is [sold for the other token](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L302-L317);
1. the token is then used to [repay the debt](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L323).

As can be seen, a portion or the entire amount of LP tokens is redeemed for the borrowed token, and the redeemed amount is then used to repay the debt. Thus, by the definition of the risk formula, the total debt cannot be repaid because the value of LP tokens will almost always be smaller than the value of the debt of a position.
## Impact
Users cannot repay their entire debt if the risk of their position is non-zero. Such positions, while being the most common kind of positions in the protocol, are forced into unavoidable liquidations. Liquidators, on the other hand, [use their own funds](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L523-L527) to repay the debt of a liquidated position.
## Code Snippet
[BlueBerryBank.sol#L483-L494](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L483-L494)
[IchiVaultSpell.sol#L297-L323](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L297-L323)
## Tool used
Manual Review
## Recommendation
Consider allowing borrowers to repay their debt from their own funds, if the value of their LP tokens is not enough to cover their debt.