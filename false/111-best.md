OCC

high

# Uncontrolled access to setRef() function

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/BandAdapterOracle.sol#L27

## Summary
`setRef() `may lead to uncontrolled access which could result in wrong prices being reported. 

## Vulnerability Detail
Although `setRef()` function can only be called by the contract owner, there is no verification to confirm that the new reference source address is a trustworthy source. If someone gains control of the contract owner's account, she/he could set the reference source to a malicious contract, which could result in incorrect prices being reported.

## Impact
High

## Code Snippet
Manually Review

## Tool used
Manually Review

### Manual Review
Done

## Recommendation
Implementing a multi-signature or time-based delay mechanism may be helpful to ensure that any modifications to the reference source address are thoroughly vetted and agreed by several parties. Or using a whitelist of trusted reference sources that the contract owner can select .