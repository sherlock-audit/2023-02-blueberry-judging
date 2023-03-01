foxb868

medium

# Manipulation of input in ways that the function was not designed to handle.

## Summary

## Vulnerability Detail
The `borrowBalanceStored`, `getPositionInfo`, and `getPositionDebtShareOf` functions take `positionId` and token as inputs, but the contract does not check whether these inputs are properly validated or not.

This can lead to various issues in such a way if the `positionId` is invalid or a non-existent position, the function will still execute without throwing any errors and may return unintended or wrong results. Anyone can take advantage of this and call the function with a fake `positionId` and obtain sensitive information or even steal funds.

## Impact
Any user could call these functions with any input value, even invalid ones, and receive a result without the contract validating that the input is correct or safe to process. 

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L270-L283
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L337-L353
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L386-L392

## Tool used

Manual Review

## Recommendation
Including input validation for the affected functions, validation ensures that the input provided is valid before executing the code, which can help prevent unintended consequences from incorrect input.

Implement a similar solution for the other vulnerable functions in the contract, including `borrowBalanceStored`, `getPositionInfo`, and `getPositionDebtShareOf`.