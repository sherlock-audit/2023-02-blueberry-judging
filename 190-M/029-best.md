cergyk

medium

# Aggregate oracle may revert even if enough valid prices are available

## Summary
In AggregatorOracle.sol, For the case of 3 base oracles, there is an algorithm to determine if there are 2 oracle prices which can be used. The algorithm reverts in a case where 2 oracles are valid.  

## Vulnerability Detail

In AggregatorOracle.sol:
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/AggregatorOracle.sol#L130-L140

in the case the oracle price at index 1 is out of bounds, but 0 and 2 are in bounds, the algorithm reverts.

## Impact
This can have a high impact if the oracle is used to price an asset of the protocol as it can lead to the revert of a liquidation call.

## Code Snippet

## Tool used

Manual Review

## Recommendation
Compare if 0 and 2 are relatively in bound.