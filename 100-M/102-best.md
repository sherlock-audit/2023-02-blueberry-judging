0Kage

high

# `isLiquidatable` function underprices risk, potentially preventing (or delaying) liquidations of undercollateralized positions

## Summary
Blueberry bank contract has a `isLiquidatable` function that checks if a given position is undercollateralized, and needs to be  liquidated. Liquidators will call this function to check if the debt + strategy losses for a given position as a proportion of posted collateral exceed the liquidation threshold.

This function computes the `risk` of a position using its debt value, collateral value and underlying value. Debt value uses the unaccrued total debt that ignores the outstanding interest accrued on `cToken` borrowings. As a result, debt value used for risk calculation is lesser than actual value. This leads to an `underpricing` of risk that can prevent or delay liquidators from initiating a liquidation action leading to under-collateralization & protocol losses.

## Vulnerability Detail
[`isLiquidatable`](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L503) function calculates risk of a given position by computing debt value, position value and underlying value in the `getPositionRisk` function.

However, debt value calculated by [`getDebtValue`](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L466) function does not include the interest accrued on `cTokens` on the outstanding debt.

```solidity
    function getDebtValue(uint256 positionId)
        public
        view
        override
        returns (uint256)
    {
        uint256 value = 0;
        Position storage pos = positions[positionId];
        uint256 bitMap = pos.debtMap;
        uint256 idx = 0;
        while (bitMap > 0) {
            if ((bitMap & 1) != 0) {
                address token = allBanks[idx];
                uint256 share = pos.debtShareOf[token];
                Bank storage bank = banks[token];
                uint256 debt = (share * bank.totalDebt).divCeil(bank.totalShare); //-audit totalDebt here does not include the accrued component since last update
                value += oracle.getDebtValue(token, debt);
            }
            idx++;
            bitMap >>= 1;
        }
        return value;
    }
```

As a result, debt value used for `risk` calculations is always lesser than or equal to actual debt outstanding. A lower risk value can generate a false signal to liquidators that a position is sufficiently collaterized. In reality, such a loan is under collateralized and should have gone for liquidation. (Note that `isLiquidatable` called within `Liquidate` function is correct value, because the function calls `poke` modifier that updates debt value)


## Impact
Incases where significant time has elapsed since last debt update, `accrued` component could be significant in comparison to total debt. While unaccrued debt value might generate a `risk` value below liquidation threshold, including accrued component might exceed that threshold. This edge case is dangerous because it gives a false comfort to liquidators that position is sufficiently collateralized.

POC:

- Say Bank has total debt of 1000 USDC
- Alice borrows 80 USDC (8%) by posting 100 USDC worth ETH as collateral
- After 30 days, say total accrued is 50 USDC
- Current Risk calculation uses Alice debt value as 80 USDC
- Actual debt value of Alice is 8% \* (1000 + 50) = 84 USDC
- All other things being equal (position losses + collateral posted), protocol underpriced risk of Alice positions

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L503

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L466

## Tool used
Manual Review

## Recommendation

Recommend replacing line 466 below 

```solidity

 uint256 debt = (share * bank.totalDebt).divCeil(bank.totalShare)

```
with following

```solidity

 uint256 debt = (share * ICErc20(bank.cToken).borrowBalanceCurrent(address(this))).divCeil(bank.totalShare)

```
This uses the latest debt value at the current timestamp for calculating risk of the position.