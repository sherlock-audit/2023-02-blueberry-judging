joestakey

medium

# A user depositing in a `softVault` takes advantage of accrued lending interests of other users.

## Summary
Because of how a `softVault` mints `cToken`, a user calling `softVault.withdraw()` leads to them stealing other users accrued lending interest

## Vulnerability Detail
When a user deposits an underlying token into a `SoftVault`, they receive a share token in exchange, proportionally to the amount they transferred in. An equivalent `cToken` amount is also minted to the `SoftVault`.

When withdrawing, the `SoftVault` calls `cToken.redeem()` to receive the underlying token amount to transfer back to the user.

The issue is that there is no account of how long a user deposited their underlying in the `SoftVault`. This mean when `withdrawing`, a user benefits from other users deposits.

E.g: Alice deposits in a `SoftVault` at block N, then calls `withdraw` at N+5. The `cToken.redeem()` call takes into account minting that were done in the past, resulting in Alice receiving more underlying token that what she should.

## Impact
A user can deposit and withdraw within a short amount of time and receive interests accrued for a much longer period by other depositors of `SoftVault`.
It is essentially unfair to early depositors.

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/vault/SoftVault.sol#L105

## Tool used
Manual Review

## Recommendation
Use additional storage variables in the vault to account for a user deposit time, to compute a fairer amount when `withdrawing`