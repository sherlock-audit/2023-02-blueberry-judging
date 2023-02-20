koxuan

medium

# disincentives for position owners that wait for position to enter liquidatable status is not implemented

## Summary
According to the docs, a position owner does not want the position to enter liquidatable status because there will be liquidation bot that will receive a cut from the position value. However, no such logic has been implemented, and hence there is no disincentive for user's position to enter liquidatable status.

## Vulnerability Detail
According to the docs from https://docs.blueberry.garden/earn/what-are-liquidations,

You want to avoid liquidation because at that time, depending on the collateral asset supplied your remaining position value would be paid to the liquidator bot as a reward for closing your position and ensuring lenders were paid back.

Looking at the `liquidate` function and other source code areas, the feature of sending the liquidator bot a slice of the remaining position value is not implemented.

```solidity
    function liquidate(
        uint256 positionId,
        address debtToken,
        uint256 amountCall
    ) external override lock poke(debtToken) {
        if (amountCall == 0) revert ZERO_AMOUNT();
        if (!isLiquidatable(positionId)) revert NOT_LIQUIDATABLE(positionId);
        Position storage pos = positions[positionId];
        Bank memory bank = banks[pos.underlyingToken];
        if (pos.collToken == address(0)) revert BAD_COLLATERAL(positionId);

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

        // Transfer position (Wrapped LP Tokens) to liquidator
        IERC1155Upgradeable(pos.collToken).safeTransferFrom(
            address(this),
            msg.sender,
            pos.collId,
            liqSize,
            ""
        );
        // Transfer underlying collaterals(vault share tokens) to liquidator
        if (
            address(ISoftVault(bank.softVault).uToken()) == pos.underlyingToken
        ) {
            IERC20Upgradeable(bank.softVault).safeTransfer(
                msg.sender,
                uVaultShare
            );
        } else {
            IERC1155Upgradeable(bank.hardVault).safeTransferFrom(
                address(this),
                msg.sender,
                uint256(uint160(pos.underlyingToken)),
                uVaultShare,
                ""
            );
        }

        emit Liquidate(
            positionId,
            msg.sender,
            debtToken,
            amountPaid,
            share,
            liqSize,
            uTokenSize
        );
    }
```

## Impact

Users are not disincentivize to wait for their position to enter liquidatable status.

## Code Snippet
https://docs.blueberry.garden/earn/what-are-liquidations
[BlueBerryBank.sol#L511-L572](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L511-L572)

## Tool used

Manual Review

## Recommendation

Recommend implementing the logic that sends a slice of the position value to the liquidator bot. 
