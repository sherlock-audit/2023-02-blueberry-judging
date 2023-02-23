rvierdiiev

high

# BlueBerryBank doesn't poke all position debt tokens, before checking getDebtValue

## Summary
BlueBerryBank doesn't poke all position debt tokens, before checking getDebtValue
## Vulnerability Detail
When user wants to borrow tokens using IchiVaultSpell, then `_validateMaxLTV` [is called](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L149).
This function will take [debtValue](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L102) and check if it's less than backed by collateral.

This is how debt value is checked.
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L451-L475
```solidity
    function getDebtValue(uint256 positionId)
        public
        view
        override
        returns (uint256)
    {
        uint256 value = 0;
        Position storage pos = positions[positionId];
        uint256 bitMap = pos.debtMap;
        uint256 idx = 0;
        while (bitMap > 0) {
            if ((bitMap & 1) != 0) {
                address token = allBanks[idx];
                uint256 share = pos.debtShareOf[token];
                Bank storage bank = banks[token];
                uint256 debt = (share * bank.totalDebt).divCeil(
                    bank.totalShare
                );
                value += oracle.getDebtValue(token, debt);
            }
            idx++;
            bitMap >>= 1;
        }
        return value;
    }
```
It will loop through `pos.debtMap` and will add borrowed amount from all banks.
For example this position has 2 banks, where it borrowed funds. `bank.totalDebt` is total debt for specific token that bank provides and during the time interests are increasing and totalDebt grows.
That's why all function inside BlueBerryBank use `poke(token)` modifier. This modifier updates `bank.totalDebt` for the bank that gives that token as a loan.

The problem is that `poke` modifier has only one token param and updates debt for this 1 token. But position can have more than one borrowed tokens and totalDebt is not updated for another tokens when user wants to get new loan. As result, because debt is not updated for another tokens, loan to value check will pass in case when it should not.

Example.
1.User borrows 100 usdc. He put some collateral, let's say 40 usdt. Suppose that maximum leverage is x5 here, so he can borrow more 100 usdc in future.
2.After some long time user wants to borrow for same position from another bank which provides dai. He takes maximum available loan and receive 100 dai. `poke` was called for dai token, but nor for usdc, where user has debt. So now he can't borrow more assets, as his collateral is used.
3.However during that time usdc loan that user took, accrued interests, so user has debt 101 usdc.
4.As result user was able to borrow more tokens, then he has collateral as all his debt tokens were not updated with new total debt.
## Impact
User can borrow more assets, then he has collateral for.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
As position can have more than 1 debt token, you need to update interests for all of them when calculating position's debt.