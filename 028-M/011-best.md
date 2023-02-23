koxuan

medium

# user cannot closePosition when borrow token is removed from whitelist

## Summary
If borrowed token is removed from collateral list, user's position cannot be closed as he is unable to repay his loan. 

## Vulnerability Detail

Notice that `onlyWhitelistedToken`  is used as a modifer, in the event that the borrow token of the position is removed from whitelist, repay will fail which means that user position cannot be closed. See code snippet for the call stack from `closePosition` to `repay`.

```solidity
    function repay(address token, uint256 amountCall)
        external
        override
        inExec
        poke(token)
        onlyWhitelistedToken(token)
    {
        if (!isRepayAllowed()) revert REPAY_NOT_ALLOWED();
        (uint256 amount, uint256 share) = repayInternal(
            POSITION_ID,
            token,
            amountCall
        );
        emit Repay(POSITION_ID, msg.sender, token, amount, share);
    }

```


## Impact

User cannot close position if the borrow token of the position is removed from whitelist. 

## Code Snippet
[IchiVaultSpell.sol#L357-L364](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L357-L364)
[IchiVaultSpell.sol#L394-L401](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L394-L401)
[IchiVaultSpell.sol#L323](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L323)
[BasicSpell.sol#L108-L113](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/BasicSpell.sol#L108-L113)
[BlueBerryBank.sol#L740-L754](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L740-L754)

## Tool used

Manual Review

## Recommendation

Recommend allowing `repay` to work for non whitelisted token so that user can close their position even when the borrowed token is removed from whitelist.
