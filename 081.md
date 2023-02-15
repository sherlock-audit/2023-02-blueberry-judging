rvierdiiev

high

# BlueBerryBank.liquidate supposes that only 1 token can be borrowed for the position

## Summary
BlueBerryBank.liquidate supposes that only 1 token can be borrowed for the position. Because of that liquidator of 1 token receives all underlying tokens.
## Vulnerability Detail
To borrow user can call execute function and provide spell and `openPosition` with parameters.
One of params that user provides [is `borrowToken`](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L169) which is the token user wants to borrow form bank.

Then later [`doBorrow` function](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L135) is called whick then calls `BlueBerryBank.borrow` function. This function will calculate shares user wants to borrow and in case if it's new borrow token for position it will [update `pos.debtMap`](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L727-L729) with new token.
Then it will just [borrow tokens from cToken](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L863) and transfer it back to spell.
It's important to note that it's possible to borrow different tokens for the position. For that reason we have `pos.debtMap` which contains banks, where user borrowed and `pos.debtShareOf` which contains shares amount that user borrowed for specific token.
So actually it's possible that user has position with 1 collateral token(usdt for example) and 2 borrowed tokens(usdc and dai for example).

Now let's check BlueBerryBank.liquidate function which allows user to liquidate position.
Liquidator provides `debtToken` and `amountCall` [params](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L513-L514), which should be used in order to repay the position.
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L522-L535
```solidity
        uint256 oldShare = pos.debtShareOf[debtToken];
        (uint256 amountPaid, uint256 share) = repayInternal(
            positionId,
            debtToken,
            amountCall
        );


        uint256 liqSize = (pos.collateralSize * share) / oldShare;
        uint256 uTokenSize = (pos.underlyingAmount * share) / oldShare;
        uint256 uVaultShare = (pos.underlyingVaultShare * share) / oldShare;


        pos.collateralSize -= liqSize;
        pos.underlyingAmount -= uTokenSize;
        pos.underlyingVaultShare -= uVaultShare;
```

The problem in this function is that it supposes that position has only 1 borrow token, but it's not like that. 
Because of if liquidator will repay full borrowed amount of any borrow token, from position borrowed tokens list, he will receive full collateral back as calculation use only shares of token, provided by liquidator.
This leads to lose of funds for protocol.

Example.
1.There are 2 banks: for usdc and dai. usdt is allowed as collateral.
2.User provides enough collateral, let's say 200 usdt in order to borrow 1 dai and 1000 usdc. He borrows that for the same position.
3.So he currently has 200 usdt collateral, 2 borrowed tokensL dai, usdc.
4.In some time this position becomes liquidatable.
5.Liquidator provides dai as `debtToken` and 1 as `amountCall` to `liquidate` function.
6.As result he repaid only 1 dai and received 200 usdt

## Impact
Lose of funds for protocol.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
Don't forget that one position can have several borrowed tokens.