berndartmueller

high

# `SoftVault` accrued interest is not refunded and stuck forever

## Summary

Withdrawing isolated collateral tokens does not refund the accrued interest from Blueberry's lending market and remains stuck forever in the `BlueBerryBank` contract.

## Vulnerability Detail

Isolated collateral is lent out to Blueberry's lending market (a Compound fork) via the `SoftVault` contract. These deposits accrue interest over time.

When a user withdraws isolated collateral using the `BlueBerryBank.withdrawLend` function within a spell, the withdrawn amount (`wAmount`) is capped at the initially deposited amount (`pos.underlyingAmount`).

Thus, if the withdrawn amount `wAmount` is greater than `pos.underlyingAmount`, the delta, which is the accrued interest, remains in the `BlueBerryBank` contract and is not refunded to the user.

## Impact

Accrued cToken interest is unrecoverable and stuck forever in the `BlueBerryBank` contract.

## Code Snippet

[BlueBerryBank.sol#L693-L695](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L693-L695)

```solidity
669: function withdrawLend(address token, uint256 shareAmount)
670:     external
671:     override
672:     inExec
673:     poke(token)
674: {
675:     Position storage pos = positions[POSITION_ID];
676:     Bank storage bank = banks[token];
677:     if (token != pos.underlyingToken) revert INVALID_UTOKEN(token);
678:     if (shareAmount == type(uint256).max) {
679:         shareAmount = pos.underlyingVaultShare;
680:     }
681:
682:     uint256 wAmount;
683:     if (address(ISoftVault(bank.softVault).uToken()) == token) {
684:         ISoftVault(bank.softVault).approve(
685:             bank.softVault,
686:             type(uint256).max
687:         );
688:         wAmount = ISoftVault(bank.softVault).withdraw(shareAmount);
689:     } else {
690:         wAmount = IHardVault(bank.hardVault).withdraw(token, shareAmount);
691:     }
692:
693:     wAmount = wAmount > pos.underlyingAmount // @audit-info Accrued Compound cToken interest is stuck from here on due to capping the `wAmount`. Interest  is unrecoverable and stuck forever
694:         ? pos.underlyingAmount
695:         : wAmount;
696:
697:     pos.underlyingVaultShare -= shareAmount;
698:     pos.underlyingAmount -= wAmount;
699:     bank.totalLend -= wAmount;
600:
701:     wAmount = doCutWithdrawFee(token, wAmount);
702:
703:     IERC20Upgradeable(token).safeTransfer(msg.sender, wAmount);
704: }
```

## Tool used

Manual Review

## Recommendation

Consider either keeping the accrued cToken interest as part of protocol revenue or refunding it to the user.
