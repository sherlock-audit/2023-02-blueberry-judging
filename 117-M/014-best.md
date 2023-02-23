koxuan

medium

# liquidate will revert if amountCall is more than debt which can lead to DOS

## Summary
A user can liquidate a liquidatable position. However, a logic in the coding allows others to frontrun liquidate tx and DOS it. 


## Vulnerability Detail

Let's say a position has 1000 debt. A user sees that the position is liquidatable, and calls liquidate with the full amount of 1000 debtToken. 
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
A griefer sees the tx in the mempool. He decides to frontrun the tx by calling `liquidate` with 1 amountCall. Now  the debt would be 999. Now notice that in `repayInternal` that if `paid > oldDebt`, the tx will revert. User can keep frontrunning liquidator and preventing him from repaying the full amount. 

```solidity
    function repayInternal( 
        uint256 positionId,
        address token,
        uint256 amountCall
    ) internal returns (uint256, uint256) {
        Bank storage bank = banks[token];
        Position storage pos = positions[positionId];
        uint256 totalShare = bank.totalShare;
        uint256 totalDebt = bank.totalDebt;
        uint256 oldShare = pos.debtShareOf[token];
        uint256 oldDebt = (oldShare * totalDebt).divCeil(totalShare);
        if (amountCall == type(uint256).max) {
            amountCall = oldDebt;
        }
        amountCall = doERC20TransferIn(token, amountCall);
        uint256 paid = doRepay(token, amountCall);
        if (paid > oldDebt) revert REPAY_EXCEEDS_DEBT(paid, oldDebt); // prevent share overflow attack
        uint256 lessShare = paid == oldDebt
            ? oldShare
            : (paid * totalShare) / totalDebt;
        bank.totalShare = totalShare - lessShare;
        uint256 newShare = oldShare - lessShare;
        pos.debtShareOf[token] = newShare;
        if (newShare == 0) {
            pos.debtMap &= ~(1 << uint256(bank.index));
        }
        return (paid, lessShare);
    }

```

```solidity
    function doRepay(address token, uint256 amountCall)
        internal
        returns (uint256 repaidAmount)
    {
        Bank storage bank = banks[token]; // assume the input is already sanity checked.
        IERC20Upgradeable(token).approve(bank.cToken, amountCall);
        if (ICErc20(bank.cToken).repayBorrow(amountCall) != 0)
            revert REPAY_FAILED(amountCall);
        uint256 newDebt = ICErc20(bank.cToken).borrowBalanceStored(
            address(this)
        );
        repaidAmount = bank.totalDebt - newDebt;
        bank.totalDebt = newDebt;
    }

```

## Impact
Adversary can cause DOS to `liquidate` by preventing liquidator from calling `liquidate` with full amount. 

## Code Snippet
[BlueBerryBank.sol#L511-L572](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L511-L572)
[BlueBerryBank.sol#L760-L787](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L760-L787)
[BlueBerryBank.sol#L877-L890](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L877-L890)



## Tool used

Manual Review

## Recommendation
If amountCall is more than debt, it should be set to debt
```solidity
if (amountCall > oldDebt){
  amountCall = oldDebt;
}
```
