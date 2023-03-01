SPYBOY

high

# User can withdraw Lending without paying withdraw-fee which will lead to financial loss

## Summary
While Lending and Withdraw-lend BlueBerryBank bank charge DepositFee and WithdrawFee. I have found the wrong implementation in which the user can withdraw lending without paying WithdrawFee. 
POC:
1) bob has deposited lend using the function `lend()`. while withdrawing lend using `withdrawLend()` function he will get charged WithdrawFee.
2) To bypass this bob can call the `liquidate()` function to withdraw lend. bob will first adjust his position in such a way that he can liquidate his position.
3) bob will withdraw lend without paying withdraw-fee. cause WithdrawFee is not applied while liquidating.
## Vulnerability Detail

## Impact
This will be a financial loss for BlueBerryBank. by using this method users can withdraw lend without paying to a withdrawal fee. also, this will be unfair to the users who are paying withdrawal fees while withdrawing lending.

## Code Snippet
withdrawLend() : https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L669-L704
liquidate() : https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L511-L572
## Tool used

Manual Review

## Recommendation
Charge WithdrawFee while liquidating position.