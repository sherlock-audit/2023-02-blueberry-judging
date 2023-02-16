0Kage

high

# `liquidate` function can be manipulated by a malicious liquidator to drain all funds of a user, both collateral and underlying tokens

## Summary
When a `debt` is repaid by a liquidator inside a `liquidate` function, liquidator is given a share of collateral and underlying token. Simultaneously, this share of collateral/underlying is reduced from owner. This `share` is currently computed solely based on the `debt` token that is being repaid by the liquidator - this can be easily manipulated by a liquidator who can repay 100% of debt token that constitutes the smallest component (by value) of the overall debt. By doing this, liquidator can stake claim on 100% of the underlying vault share & collateral (see POC) causing a total loss to the position owner.

## Vulnerability Detail
`liquidate` function repays a `debt` token and gets a share of `collateral` and `underlyingVaultShare`. Reduction in the owner share is calculated in the snippet below

```solidity

        uint256 oldShare = pos.debtShareOf[debtToken];
        (uint256 amountPaid, uint256 share) = repayInternal(
            positionId,
            debtToken,
            amountCall
        );

        uint256 liqSize = (pos.collateralSize * share) / oldShare;
        uint256 uTokenSize = (pos.underlyingAmount * share) / oldShare;
        uint256 uVaultShare = (pos.underlyingVaultShare * share) / oldShare;

        pos.collateralSize -= liqSize;
        pos.underlyingAmount -= uTokenSize;
        pos.underlyingVaultShare -= uVaultShare;

```
Although `collateralSize` and `underlyingVaultShare` back the entire `debt` of the owner across multiple banks, notice that this single debt token is deciding how much of share needs to be distributed to the liquidator. Malicious user will always pick the lowest representation `debt` token & pay off the entire 100% of it. Since the share paid by liquidator is 100%,  current implementation allows her to claim the entire 100% of `underlyingVaultShare` and collateral.

## Impact
Reducing underlying & collateral share based on a single `debt` token repayment creates a high risk vulnerability where a position owner can lose all his funds.

POC

- Alice has underlying of `1000 USDC`, collateral of `1 ETH` (valued at say `1500 USDC`)
- Alice has borrowed 600 USDC from `USDC` bank (let's say her debt share in this bank = 10%) & 0.1 ETH from `ETH` bank (valued at `150 USDC`). Let's assume her debt share for `ETH` bank is 1%.

- Lets assume loan is currently liquidatable
- Malicious liquidator repays 100% of lowest valued debt component, which in this case `0.1 ETH`
- `repayInternal` function returns `share` of `1%`, `oldShare` also is `1%`
- share reduction just by using `ETH` token repayment is 100% since all the ETH borrowing is repaid by liquidator
- Entire collateral (that backs both 600 USDC & 0.1 ETH) is transferred to liquidator just by repaying `0.1` ETH
- Additionally, entire `underlyingVaultShare` representing `1000 USDC` of underlying is transferred to liquidator

In above case, correct implementation should be

```
oldShare = 10% (USDC) * price_USDC/USD + 1% (ETH) * price_ETH/USD
newShare = 10% (USDC) * price_USDC/USD + 0% (ETH) * price_ETH/USD
lessShare = oldShare - newShare
```

Collateral to liquidator:
```
collateral  = (lessShare / oldShare) * pos.collateralSize
```
## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L511

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L780

## Tool used
Manual Review

## Recommendation

Current implementation is only using `new` and `old` shares of specific debt token that is repaid.

Correct implementation has to change to following:

    - `oldShare` is the `SUM` of all debtShares across all debt tokens (based on `debtMap` of positions)
    - `newShare` is the `SUM` of all debtShares across all debt tokens (AFTER liquidator pays a given debt token)

In the above case, shares are representation of entire `debtMap` and not a single debt.