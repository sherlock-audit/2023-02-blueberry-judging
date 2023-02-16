SPYBOY

high

# User can repay `Blacklisted` Tokens

## Summary
In the `BlueBerryBank.sol` contract there is a function `repay()` to repay the borrowed amount.  before executing repay() function there is a modifier called `onlyWhitelistedToken`. which checks whether the token user is repaying is whitelisted or not. I found the way through which users can repay backlisted tokens and can withdraw collateral.
POC:
1) Bob has borrowed lots of `xyz` tokens from BlueBerryBank. 
2) After some time, something wrong happens with that token (maybe that token gets the rug pulled/hacked).
3) Admin decided to blacklist `xyz` token using the function `whitelistTokens()`.
4) Now bob will try to repay that tokens but because of the `onlyWhitelistedToken` modifier bob will not be able to repay the `xyz` tokens.
5) To bypass this bob can call the `liquidate()` function to liquidate his position. because the `onlyWhitelistedToken` modifier is not applied to the `liquidate()` function.  although bob may suffer from some percentage of loss he can repay failed `xyz` tokens and can recover his collateral. 

## Vulnerability Detail

## Impact
Although BlueBerryBank doesn't want to get repaired for a specific token. users can still repay those tokens through `liquidate()` function.  BlueBerryBank will not be able to recover those losses through collaterals.

## Code Snippet
repay: https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L740-L754
whitelistTokens: https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L169-L181
liquidate: https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L511-L572


## Tool used

Manual Review

## Recommendation
Add the `onlyWhitelistedToken` modifier to `liquidate()` function
```solidity
function liquidate(uint256 positionId, address debtToken, uint256 amountCall ) external override lock onlyWhitelistedToken poke(debtToken){
//rest of code
...
}
```