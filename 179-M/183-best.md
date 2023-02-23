mahdikarimi

medium

# wrong check prevents to deposit funds in softvault

## Summary
wrong check prevents to deposit funds in softvault
## Vulnerability Detail
There is a check in SoftVault that ensures minted cToken amount is not zero but it uses wrong check and if deposit amount and minted cTokens is not zero reverts the transaction . 
`if (cToken.mint(uBalanceAfter - uBalanceBefore) != 0)
            revert LEND_FAILED(amount);`
## Impact
Users unable to deposit funds in softvault 
## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/vault/SoftVault.sol#L79-L80
## Tool used

Manual Review

## Recommendation
you can use this check instead 
`if (cToken.mint(uBalanceAfter - uBalanceBefore) == 0);`