Jeiwan

medium

# `getPositionIdsByOwner` may be unavailable, breaking integrations

## Summary
The `getPositionIdsByOwner` function of `BlueBerryBank` may consume too much gas. Integrations that use the function may not be able to find all positions of an owner.
## Vulnerability Detail
The [getPositionIdsByOwner](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L315) function returns the list of positions controlled by a user. For that, the function iterates over all position IDs and check the owner of every position:
```solidity
uint256[] memory matchingIds = new uint256[](nextPositionId);
uint256 index;
for (uint256 i = 0; i < nextPositionId; i++) {
    if (positions[i].owner == owner) {
        matchingIds[index] = i;
        index++;
    }
}
```

However, this unbounded iteration may eventually consume the entire gas limit of a block, causing a revert in a transaction that executes it. Since the `nextPositionId` variable is the [global counter of opened positions](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L585), over the lifespan of the contract, it may become a high enough number to make the `getPositionIdsByOwner` function consume too much gas. In the extreme case, the amount of gas required by the function can be as big as the gas limit of a block–calling the function in this case will require consuming the entire gas limit of a block, causing any contract that calls the function to revert.
## Impact
The `getPositionIdsByOwner` may consume too much gas and be too expensive to call. In the worst case, the function may consume the entire gas limit of a block, causing reverts.
## Code Snippet
[BlueBerryBank.sol#L322-L327](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L322-L327)
## Tool used
Manual Review
## Recommendation
In addition to the `positions` mapping, consider adding another mapping that will map owners to their position.