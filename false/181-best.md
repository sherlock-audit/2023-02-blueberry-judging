mahdikarimi

high

# wrong check in softvault prevents from withdraw

## Summary
wrong check prevents funds to be withdrawn in softvault
## Vulnerability Detail
There is a check in SoftVault that ensures redeemed amount is not zero but it uses wrong check and if some amount redeemed reverts the transaction . 
`if (cToken.redeem(shareAmount) != 0) revert REDEEM_FAILED(shareAmount);`
## Impact
Users unable to withdraw their funds  
## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/vault/SoftVault.sol#L105
## Tool used

Manual Review

## Recommendation
you can use this check instead 
`if (cToken.redeem(shareAmount) == 0) revert REDEEM_FAILED(shareAmount);`