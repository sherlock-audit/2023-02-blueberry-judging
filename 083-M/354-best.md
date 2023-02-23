minhtrng

high

# Excess funds when withdrawing from lending stuck permanently

## Summary

Excess funds returned when withdrawing from a lend postion get stuck in the bank permanently.

## Vulnerability Detail

When withdrawing from a lending position in `BlueBerryBank.withdrawLend` any amounts withdrawn from the vaults are capped at the underlying amount of the position:

```js
wAmount = ISoftVault(bank.softVault).withdraw(shareAmount);
...
wAmount = wAmount > pos.underlyingAmount
    ? pos.underlyingAmount
    : wAmount;
...
IERC20Upgradeable(token).safeTransfer(msg.sender, wAmount);
```

Excess amounts can occur when the underlying amount was deposited into the SoftVault, as it further puts the tokens into Blueberrys Compound fork, where they accrue value for being lent out. The problem is, that the current implementation does not process the excess funds, nor is there another way to transfer them out of the bank, hence they are stuck permanently once withdrawn from the SoftVault.

## Impact

Permanent lock of partial funds.

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/03048217af4ea396b3d8f96cb8edb3b23f0c9b47/contracts/BlueBerryBank.sol#L688-L703

## Tool used

Manual Review

## Recommendation
From a business level perspective it would make sense to send the full amount withdrawn from the SoftVault to the user, as those are the pro rata amounts that their shares were worth. Alternatively, calculate the exact excess amount and transfer them somewhere else, e.g. treasury.