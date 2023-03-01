Jeiwan

medium

# Users may not repay their debt partially when above the maximal LTV

## Summary
The protocol doesn't allow users to repay their debt partially if the LTV after the repayment is above the maximal LTV of the strategy. The liquidation threshold might still not be reached in such situations, however users cannot bring their LTV slightly below the liquidation threshold.
## Vulnerability Detail
When closing a position and repaying debt, there's a [check that requires the position's LTV to be below the maximal LTV of the strategy](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L325). While this check [protects against excessive borrowings](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L149), it shouldn't disallow users to repay their debt, even if no new borrowings are allowed at their current LTV.

The protocol defines two kinds of thresholds: the [maximal LTV](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L95), which sets the [maximal debt-to-collateral ratio when opening a position](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L101); the [liquidation threshold](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/CoreOracle.sol#L63-L67) which sets the debt-to-collateral ratio at which a position [can be liquidated](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L504). Users' position may drift above the maximal LTV due to accrued borrow interests but still remain below the liquidation threshold. And users may wish to maintain their positions above the maximal LTV and below the liquidation threshold. However, the debt repayment mechanism doesn't allow them to do that the due to the [maximal LTV check](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L325) in the [withdrawInternal](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L276) function.

Disallowing partial repayments negative consequences for the protocol: unpaid debt may keep increasing, leading to bad debt. Small debt repayments may help users to earn yield and stay below the liquidation threshold.
## Impact
Users cannot repay debt partially while staying below the liquidation threshold, potentially leading to liquidations and/or more bad debt for the protocol.
## Code Snippet
[IchiVaultSpell.sol#L325](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L325)
## Tool used
Manual Review
## Recommendation
In the `withdrawInternal` function, consider checking current position's LTV against the liquidation thresholds, instead of the maximal LTV.