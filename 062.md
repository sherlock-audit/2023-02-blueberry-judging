csanuragjain

medium

# Withdrawal fee bypass on Liquidation

## Summary
On withdrawing lent token, a withdraw fees is deducted. But seems that on Liquidation, withdrawal fees is not deducted from the final amount. This is not expected and if the fees is large and User can liquidate his position himself in order to avoid withdrawal fees

## Vulnerability Detail

1. Observe the [withdrawLend](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L669) function

```solidity
function withdrawLend(address token, uint256 shareAmount)
        external
        override
        inExec
        poke(token)
    {
        ...

        wAmount = doCutWithdrawFee(token, wAmount);

        IERC20Upgradeable(token).safeTransfer(msg.sender, wAmount);
    }
```

2. A withdrawal fees is deducted from the final amount
3. Lets see if this withdrawal fees is also deducted if user position get liquidated which is done using [liquidate](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L511) function

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

4. As we can see their is no deduction of withdrawal fee

## Impact
User will skip giving withdrawal fees on liquidation, which gives loss to protocol

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L511-L572

## Tool used
Manual Review

## Recommendation
Add withdrawal fees in liquidation logic as well