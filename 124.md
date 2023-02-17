Saeedalipoor01988

medium

# doRepay function can be DOSed

## Summary
in the BlueBerryBank contract, we use the doRepay function to repay money to the Ctoken.
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L883

repayBorrow function from token:
https://github.com/compound-finance/compound-protocol/blob/a3214f67b73310d547e00fc578e8355911c9d376/contracts/CErc20.sol#L91

## Vulnerability Detail
let's say there are conditions and the bank need to repay all debts to CToken. all deb is 1000. we call doRepay with amountCall = 1000. and this function will call repayBorrow with repayAmount = 1000, and repayBorrow will call repayBorrowInternal with repayAmount = 1000. 

an attacker can see it and front-run to repay a single token for Bank contract debt with the calling function repayBorrowBehalf (for example 1 DAI, a single token is worth almost nothing). 

https://github.com/compound-finance/compound-protocol/blob/a3214f67b73310d547e00fc578e8355911c9d376/contracts/CErc20.sol#L102

now in repayBorrowFresh
https://github.com/compound-finance/compound-protocol/blob/a3214f67b73310d547e00fc578e8355911c9d376/contracts/CToken.sol#L643

uint accountBorrowsPrev = borrowBalanceStoredInternal(borrower) = 999, and repayAmount is 1000 ! as we called  doRepay with amountCall = 1000.

now we will get revert because of this line :
 uint accountBorrowsNew = accountBorrowsPrev - actualRepayAmount = 999 - 1000 !
https://github.com/compound-finance/compound-protocol/blob/a3214f67b73310d547e00fc578e8355911c9d376/contracts/CToken.sol#L679

## Impact
So, in critical market conditions when the price moves quickly, DOS in doRepay can cause the liquidation of the bank's assets.

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L883

## Tool used

Manual Review

## Recommendation
use DOS protection.