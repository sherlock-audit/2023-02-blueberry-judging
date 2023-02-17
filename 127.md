obront

high

# Liquidator can take all collateral and underlying tokens for a fraction of the correct price

## Summary

When performing liquidation calculations, we use the proportion of the individual token's debt they pay off to calculate the proportion of the liquidated user's collateral and underlying tokens to send to them. In the event that the user has multiple types of debt, the liquidator will be dramatically overpaid.

## Vulnerability Detail

When a position's risk rating falls below the underlying token's liquidation threshold, the position becomes liquidatable. At this point, anyone can call `liquidate()` and pay back a share of their debt, and receive a propotionate share of their underlying assets.

This is calculated as follows:
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

// ...transfer liqSize wrapped LP Tokens and uVaultShare underlying vault shares to the liquidator
}
```
To summarize:
- The liquidator inputs a debtToken to pay off and an amount to pay
- We check the amount of debt shares the position has on that debtToken
- We call `repayInternal()`, which pays off the position and returns the amount paid and number of shares paid off
- We then calculate the proportion of collateral and underlying tokens to give the liquidator
- We adjust the liquidated position's balances, and send the funds to the liquidator

The problem comes in the calculations. The amount paid to the liquidator is calculated as:
```solidity
uint256 liqSize = (pos.collateralSize * share) / oldShare
uint256 uTokenSize = (pos.underlyingAmount * share) / oldShare;
uint256 uVaultShare = (pos.underlyingVaultShare * share) / oldShare;
```
These calculations are taking the total size of the collateral or underlying token. They are then multiplying it by `share / oldShare`. But `share / oldShare` is just the proportion of that one type of debt that was paid off, not of the user's entire debt pool.

Let's walk through a specific scenario of how this might be exploited:
- User deposits 1mm DAI (underlying) and uses it to borrow $950k of ETH and $50k worth of ICHI (11.8k ICHI)
- Both assets are deposited into the ETH-ICHI pool, yielding the same collateral token
- Both prices crash down by 25% so the position is now liquidatable (worth $750k)
- A liquidator pays back the full ICHI position, and the calculations above yield `pos.collateralSize * 11.8k / 11.8k` (same calculation for the other two formulas)
- The result is that for 11.8k ICHI (worth $37.5k after the price crash), the liquidator got all the DAI (value $1mm) and LP tokens (value $750k) 

## Impact

If a position with multiple borrows goes into liquidation, the liquidator can pay off the smallest token (guaranteed to be less than half the total value) to take the full position, stealing funds from innocent users.

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L511-L572

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L760-L787

## Tool used

Manual Review

## Recommendation

Adjust these calculations to use `amountPaid / getDebtValue(positionId)`, which is accurately calculate the proportion of the total debt paid off.