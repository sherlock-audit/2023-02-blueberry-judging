sakshamguruji

medium

# getPrice MIGHT GIVE INCORRECT RESULT DUE TO NO DECIMAL CHECK

## Summary

The getPrice function here https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/IchiLpOracle.sol#L19 might produce incorrect value for the price returned in cases
where `t0Decimal` != `t1Decimal`

## Vulnerability Detail

Assume `t0Decimal` has 18 decimals and `t1Decimal` has 22 decimals , r0 = 100 = r1 , px0 = px1 = 100. Now the contract assumes we are only dealing with 18 decimal tokens or similar decimal tokens , there fore the value calculated for `totalReserve` would be 
(100*100)/1e18 + (100*100)/1e22 which would give us 1e-14+ 1e-18 instead of the expected 2 * (1e-14).

## Impact

Wrong price returned by the function getPrice

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/IchiLpOracle.sol#L19-L38

## Tool used

Manual Review

## Recommendation

Check if t0decimal == t1decimal