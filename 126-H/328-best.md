berndartmueller

high

# A liquidator can repay the smaller debt position to fully liquidate a position and receive the full collateral

## Summary

An incorrect full liquidation of a position can occur by repaying only the token with the smaller debt, enabling a liquidator to receive the entire collateral and the protocol to accrue bad debt.

## Vulnerability Detail

When a liquidatable position with multiple borrowed tokens (e.g. USDC and ICHI) is due for liquidation, a liquidator can repay the debt of the token with the smaller debt to fully liquidate the position and gain all wrapped LP tokens and isolated collateral.

This issue is caused by using the repaid `share` to determine the amount of collateral to be returned to the liquidator. `share` represents the repaid share of the given debt token (`debtToken`) and not the share of the total debt of the position.

Contrary to the Alpha Homora protocol, which uses `amountPaid` instead of `share` (see [HomoraBank.sol#L459-L465](https://github.com/AlphaFinanceLab/alpha-homora-v2-contract/blob/f74fc460bd614ad15bbef57c88f6b470e5efd1fd/contracts/HomoraBank.sol#L459-L465)).

### Test case

To demonstrate this issue, please use the provided test case at https://gist.github.com/berndartmueller/d34ce2b3708975e6b8271244cd586dfd.

Copy the test file into `test/liquidate.test.ts` and run `yarn hardhat test --grep "should be able to liquidate full position by only repaying smaller debt token"`.

The test case demonstrates how a liquidator can just repay the smaller debt position (from a user who borrowed USDC and ICHI) to liquidate the position fully and receives all of the wrapped LP token collateral as well as the isolated collateral.

## Impact

The incorrect full liquidation of a position by repaying only the smaller debt token causes the protocol to accrue bad debt.

## Code Snippet

[BlueBerryBank.sol#L523-L527](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L523-L527)

```solidity
511: function liquidate(
512:     uint256 positionId,
513:     address debtToken,
514:     uint256 amountCall
515: ) external override lock poke(debtToken) {
516:     if (amountCall == 0) revert ZERO_AMOUNT();
517:     if (!isLiquidatable(positionId)) revert NOT_LIQUIDATABLE(positionId);
518:     Position storage pos = positions[positionId];
519:     Bank memory bank = banks[pos.underlyingToken];
520:     if (pos.collToken == address(0)) revert BAD_COLLATERAL(positionId);
521:
522:     uint256 oldShare = pos.debtShareOf[debtToken];
523:     (uint256 amountPaid, uint256 share) = repayInternal( // @audit-info `share` is only the repaid share of the given `debtToken` token
524:         positionId,
525:         debtToken,
526:         amountCall
527:     );
528:
529:     uint256 liqSize = (pos.collateralSize * share) / oldShare;
530:     uint256 uTokenSize = (pos.underlyingAmount * share) / oldShare;
531:     uint256 uVaultShare = (pos.underlyingVaultShare * share) / oldShare;
532:
533:     pos.collateralSize -= liqSize;
534:     pos.underlyingAmount -= uTokenSize;
535:     pos.underlyingVaultShare -= uVaultShare;
536:
537:     // Transfer position (Wrapped LP Tokens) to liquidator
538:     IERC1155Upgradeable(pos.collToken).safeTransferFrom(
539:         address(this),
540:         msg.sender,
541:         pos.collId,
542:         liqSize,
543:         ""
544:     );
545:     // Transfer underlying collaterals(vault share tokens) to liquidator
546:     if (
547:         address(ISoftVault(bank.softVault).uToken()) == pos.underlyingToken
548:     ) {
549:         IERC20Upgradeable(bank.softVault).safeTransfer(
550:             msg.sender,
551:             uVaultShare
552:         );
553:     } else {
554:         IERC1155Upgradeable(bank.hardVault).safeTransferFrom(
555:             address(this),
556:             msg.sender,
557:             uint256(uint160(pos.underlyingToken)),
558:             uVaultShare,
559:             ""
560:         );
561:     }
562:
563:     emit Liquidate(
564:         positionId,
565:         msg.sender,
566:         debtToken,
567:         amountPaid,
568:         share,
569:         liqSize,
570:         uTokenSize
571:     );
572: }
```

## Tool used

Manual Review

## Recommendation

Consider using `amountPaid` instead of `share` to determine the amount of collateral for the liquidator.
