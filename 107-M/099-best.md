SPYBOY

high

# Incorrect amount in `bank.totalLend` after liquidating  position

## Summary
when the user deposit lending using the function `lend()` it increments `pos.underlyingToken` `pos.underlyingAmount` and `bank.totalLend` .  when the user withdraws lend using the function `withdrawLend()`. it decrements `pos.underlyingVaultShare`  `pos.underlyingAmount` and `bank.totalLend`.  when the position reaches an unhealthy state someone will liquidate that position using the function `liquidate()`  But while liquidating it only decrements `pos.underlyingAmount`, `pos.underlyingVaultShare`. although lending has been withdrawn using the `liquidate()` function `bank.totalLend` variable will be storing the old amount.  Because `bank.totalLend` is not decremented in `liquidate()`.

increments bank.totalLend in `lend()` : https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L659
decrements bank.totalLend in `withdrawLend()`: https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L697-L699
Not decrements bank.totalLend in `liquidate()` : https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L533-L535
## Vulnerability Detail

## Impact
according to functionality `bank.totalLend` should store the accurate amount of totalLend after liquidating the position because of the wrong implementation of `liquidate()` . `bank.totalLend` will be storing wrong amount after liquidation.
## Code Snippet
increments bank.totalLend in `lend()` : https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L659
decrements bank.totalLend in `withdrawLend()`: https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L697-L699
Not decrements bank.totalLend in `liquidate()` : https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L533-L535

## Tool used

Manual Review

## Recommendation
 decrement bank.totalLend in liquidate function.
```solidity
    function liquidate( uint256 positionId,  address debtToken, uint256 amountCall) external override lock poke(debtToken) {
     ...
        pos.collateralSize -= liqSize;
        pos.underlyingAmount -= uTokenSize;
        pos.underlyingVaultShare -= uVaultShare;
        bank.totalLend -= uTokenSize;     //solution
    ....
}
```