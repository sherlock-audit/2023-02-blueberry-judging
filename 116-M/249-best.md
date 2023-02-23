HonorLt

medium

# A change in whitelist can block repays

## Summary

It becomes impossible to repay a debt if the token is removed from the whitelist, leading to the risk of liquidation.

## Vulnerability Detail

`repay` function has an `onlyWhitelistedToken` modifier:
```solidity
    function repay(address token, uint256 amountCall)
        external
        override
        inExec
        poke(token)
        onlyWhitelistedToken(token)
```
This modifier is also enforced on `lend` and `borrow` actions. The problem with this is it might be possible that when the user borrowed the token it was included in the whitelist. However, later due to some reason, this token is removed from the whitelist leaving borrowers no options to discard their debt but worse the liquidations can still be performed because it does not have this whitelist requirement:
```solidity
    function liquidate(
        uint256 positionId,
        address debtToken,
        uint256 amountCall
    ) external override lock poke(debtToken)
```

Interestingly, the `withdrawLend` function does not have this modifier when it might make sense to suspend withdrawals if the token is in bad shape.

Similarly, `isRepayAllowed` can block repays, but that's another function that is depending on the grace of owners, thus I am not raising another issue.

## Impact

If the token is removed from the whitelist, the borrowers are risking getting liquidated without the ability to try repaying the debt first.

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L740-L745

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L511-L515

## Tool used

Manual Review

## Recommendation

If such a situation happens, borrowers should be able to repay old debt but not make new borrows.
