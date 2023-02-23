rvierdiiev

medium

# BlueBerryBank.getPositionIdsByOwner will stop working with out of gass error when a lot of positions will be created

## Summary
BlueBerryBank.getPositionIdsByOwner will stop working with out of gass error when a lot of positions will be created. As result frontend will be not able to show user his positions.
## Vulnerability Detail
BlueBerryBank.getPositionIdsByOwner iterates over all positions and check if owner is needed user.
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L315-L333
```solidity
    function getPositionIdsByOwner(address owner)
        external
        view
        returns (uint256[] memory ids)
    {
        uint256[] memory matchingIds = new uint256[](nextPositionId);
        uint256 index;
        for (uint256 i = 0; i < nextPositionId; i++) {
            if (positions[i].owner == owner) {
                matchingIds[index] = i;
                index++;
            }
        }


        ids = new uint256[](index);
        for (uint256 i = 0; i < index; i++) {
            ids[i] = matchingIds[i];
        }
    }
```

When big number of positions will be created this function will stop working with out of gas error.
Because of that frontend will not be able to call it and receive positions for the user. So part of UI functionality will not work.
## Impact
BlueBerryBank.getPositionIdsByOwner will stop working
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
Store all positions for user in mapping or use indexes in this function.