tives

medium

# Unbounded loop in getPositionIdsByOwner

## Summary
Unbounded loop in getPositionIdsByOwner.

## Vulnerability Detail
`getPositionIdsByOwner` loops through all positions. This can hit the block gas limit and it would revert.

## Impact
You can imagine other projects relying on this function to retrieve positions. And if they cannot retrieve the positions then users might think their funds are lost. Then users do forget about them and funds are lost.

This could also lead to a DoS attack where adversary creates many 1 wei positions and blocks this function.

## Code Snippet

```solidity
function getPositionIdsByOwner() {
	for (uint256 i = 0; i < nextPositionId; i++) {
	    if (positions[i].owner == owner) {
	        matchingIds[index] = i;
	        index++;
	    }
	}
```
[link](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol/#L315)

## Tool used
Manual Review

## Recommendation

Remove this method or add paging option so it is not unbounded
