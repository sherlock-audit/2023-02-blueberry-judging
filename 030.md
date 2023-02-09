cergyk

medium

# Not set liquidationThreshold in coreOracle does not prevent openPosition and can lead to unjust liquidations

## Summary
If `liquidationThreshold` is not set as a setting for a token on coreOracle, a user can still open an initially solvent position (risk == 0), but which becomes instantly liquidatable when risk != 0. 

## Vulnerability Detail
liquidationThreshold is retrieved from oracle to evaluate liquidatability of a position:
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/CoreOracle.sol#L197-L199

if it is not set by the admin for a token, a user can still register his position, as it may be initially healthy:
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L491
if risk == 0

## Impact

## Code Snippet

## Tool used

Manual Review

## Recommendation
Since liquidationThreshold cannot be lower than MIN_LIQ_THRESHOLD, return MIN_LIQ_THRESHOLD as a default value, or revert when trying to lend with a token which has no `liqThreshold` set