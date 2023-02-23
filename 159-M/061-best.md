rvi

medium

# execute will fail if msg.value is not 0

## Summary

## Vulnerability Detail
`execute` function is made `payable` but SPELL variable is not, so any call with non zero value to address SPELL will fail 

## Impact
`execute` function all ways will fail on calls with value higher then zero
payable functions that require ether in spells do not work (reverts)

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L595

## Tool used

Manual Review

## Recommendation
- turning `address public override SPELL` to `address public payable override SPELL`
- `address spell`  should be `address payable spell` in the `execute` parameters

OR

- converting `SPELL` to payable before calling it in `execute` function