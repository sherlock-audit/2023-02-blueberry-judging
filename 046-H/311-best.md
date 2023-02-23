GimelSec

high

# Lenders didn't receive their interest.

## Summary

The withdraw amount in `BlueBerryBank.withdrawLend()` cannot exceed `pos.underlyingAmount`. Thus, lenders cannot receive any interest. And they will lose the fund if `withdrawFee` or `depositFee` is not zero.

## Vulnerability Detail

The withdraw amount in `BlueBerryBank.withdrawLend()` cannot exceed `pos.underlyingAmount`.
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L693
```solidity
    function withdrawLend(address token, uint256 shareAmount)
        external
        override
        inExec
        poke(token)
    {
        …

        wAmount = wAmount > pos.underlyingAmount
            ? pos.underlyingAmount
            : wAmount;

        pos.underlyingVaultShare -= shareAmount;
        pos.underlyingAmount -= wAmount;
        bank.totalLend -= wAmount;

        wAmount = doCutWithdrawFee(token, wAmount);

        IERC20Upgradeable(token).safeTransfer(msg.sender, wAmount);
    }
```

And lenders will lose the fund if `withdrawFee` or `depositFee` is not zero.

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L701
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L642
```solidity
    function lend(address token, uint256 amount)
        external
        override
        inExec
        poke(token)
        onlyWhitelistedToken(token)
    {
        …
        amount = doCutDepositFee(token, amount);
        pos.underlyingToken = token;
        pos.underlyingAmount += amount;

        …
    }
```

## Impact

Lenders cannot receive any interest. And they will lose the fund if `withdrawFee` or `depositFee` is not zero.

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L693
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L701
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L642


## Tool used

Manual Review

## Recommendation

`withdrawLend` should be modified like this:

```solidity
    function withdrawLend(address token, uint256 shareAmount)
        external
        override
        inExec
        poke(token)
    {
        …
        IERC20Upgradeable(token).safeTransfer(msg.sender, doCutWithdrawFee(token, wAmount));
        wAmount = wAmount > pos.underlyingAmount
            ? pos.underlyingAmount
            : wAmount;

        pos.underlyingVaultShare -= shareAmount;
        pos.underlyingAmount -= wAmount;
        bank.totalLend -= wAmount;

    }
```
