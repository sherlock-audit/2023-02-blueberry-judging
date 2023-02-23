saian

medium

# Tokens will be stuck in the contract

## Summary

Excess tokens withdrawn from SoftVault will be stuck in the contract

## Vulnerability Detail

In BlueBerryBank#withdrawLend, the position's shares are redeemed and underlying token is transferred to the spell. If the withdrawn amount is greater than the position's total underlying, only the position's balance is transferred, the excess tokens are stuck in the contract.

## Impact

The tokens are stuck in the contract and cannot be withdrawn

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L693

```solidity
    function withdrawLend(address token, uint256 shareAmount)
        external
        override
        inExec
        poke(token)
    {
        ...

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
## Tool used

Manual Review

## Recommendation

Transfer the entire withdrawn amount to the spell

```solidity
        if(wAmount > pos.underlyingAmount) {
            pos.underlyingAmount = 0;
        } else {
            pos.underlyingAmount -= wAmount;
        }

        pos.underlyingVaultShare -= shareAmount;
        bank.totalLend -= wAmount;
```