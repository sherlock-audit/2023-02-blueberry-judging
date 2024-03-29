0xbrett8571

high

# Out-of-bounds Access Due to failure on Input Validation.

## Summary
There is no proper input to validate some of the contract functions. which all require input validation, First, there is no check on the positionId argument in the `borrowBalanceStored`, `getPositionInfo`, and `getPositionDebtShareOf` functions to ensure that the `positionId` is within the bounds of the positions array and this can lead to an out-of-bounds access flaw.

## Vulnerability Detail
Some functions do not properly validate the input arguments. Specifically, in the `borrowBalanceStored`, `getPositionInfo`, and `getPositionDebtShareOf` functions do not validate the "positionId" argument to ensure that it is within the bounds of the positions array.
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L289-L296

The `positionId` argument is used to index into the positions array to retrieve the debt share for a particular position but there is no check to ensure that the positionId is within the bounds of the positions array, which can result in accessing an invalid memory location. In this case, possible unintended consequences such as reading or modifying sensitive data or causing the contract to fail with an exception will happen.

A similar thing exists in the `getPositionInfo` function, where the positionId argument is used to index into the positions array to retrieve information about a particular position:
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L337-L361

And finally, the `getPositionDebtShareOf` function retrieves the debt share of a particular position and bank token:
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L386-L392

In both cases, I can see there is no check to ensure the positionId argument is within the bounds of the positions array, and this can result in an out-of-bounds access .

## Impact
Input validation failure on the positionId argument in a function can allow an attacker to provide a positionId outside the bounds of the positions array, which will result in an out-of-bounds access. This flaw can lead to unintended consequences, such as arbitrary code execution, denial-of-service, or even data corruption.

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L337-L361
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L386-L392

## Tool used

Manual Review

## Recommendation
I will recommend adding an input validation on the `positionId` argument in the `borrowBalanceStored`, `getPositionInfo`, and `getPositionDebtShareOf` functions to ensure that the `positionId` is within the bounds of the positions array. This can be done by adding a check to verify that the `positionId` is less than `nextPositionId` before accessing the positions array.


This will ensure that the contract will revert with an error message if an invalid `positionId` is passed as an argument, thus preventing an out-of-bounds access vulnerability.
And here's an updated implementation for the input validation:
```solidity
function borrowBalanceStored(uint256 positionId, address token)
    external
    override
    returns (uint256)
{
+   require(positionId < nextPositionId, "ERROR: Position id out of bounds");
+   return positions[positionId].borrowBalanceOf[token];
}

function getPositionInfo(uint256 positionId)
    public
    view
    override
    returns (
        address owner,
        address underlyingToken,
        uint256 underlyingAmount,
        uint256 underlyingVaultShare,
        address collToken,
        uint256 collId,
        uint256 collateralSize,
        uint256 risk
    )
{
+  require(positionId < nextPositionId, "ERROR: Position id out of bounds");
    Position storage pos = positions[positionId];
    owner = pos.owner;
    underlyingToken = pos.underlyingToken;
    underlyingAmount = pos.underlyingAmount;
    underlyingVaultShare = pos.underlyingVaultShare;
    collToken = pos.collToken;
    collId = pos.collId;
    collateralSize = pos.collateralSize;
    risk = getPositionRisk(positionId);
}

function getPositionDebtShareOf(uint256 positionId, address token)
    external
    view
    returns (uint256)
{
+  require(positionId < nextPositionId, "ERROR: Position id out of bounds");
    return positions[positionId].debtShareOf[token];
}
```
