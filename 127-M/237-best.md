Ch_301

medium

# The protocol will lose the fees of the liquidated Position with WIchiFarm

## Summary
Any **PositionFarm** will pay some [RewardsFee](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L391) to the [treasury](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/ProtocolConfig.sol#L23) for each interaction with [closePositionFarm()](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L367-L405) 

## Vulnerability Detail
If the position is a **PositionFarm**, the liquidator will invoke  [liquidate()](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L511-L572) by paying the debt for the owner and take the collateral. 
This logic will **Transfer** all the `WIchiFarm` to the liquidator 
```solidity
        // Transfer position (Wrapped LP Tokens) to liquidator
        IERC1155Upgradeable(pos.collToken).safeTransferFrom(
            address(this),
            msg.sender,
            pos.collId,
            liqSize,
            ""
        );
```
then he will invoke [burn()](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/wrapper/WIchiFarm.sol#L116-L150) to receive **ICHI Vault LP** + All Rewards from **ICHI Farm**.
## Impact
The liquidator will gain all the Rewards from ICHI Farm.

## Code Snippet

## Tool used

Manual Review

## Recommendation
**Cut the RewardsFee** on `WIchiFarm.sol`
