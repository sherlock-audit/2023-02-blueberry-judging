sinarette

high

# Harvested ICHI tokens are not collected

## Summary

````IchiVaultSpell#openPositionFarm collects vault tokens and ICHI farming rewards from ````wIchiFarm````. The collected vault tokens are re-deposited to ````wIchiFarm````, but the ICHI farming rewards are not collected from the spell contract.

## Vulnerability Detail

```solidity
        if (collSize > 0) {
            (uint256 decodedPid, ) = wIchiFarm.decodeId(collId);
            if (farmingPid != decodedPid) revert INCORRECT_PID(farmingPid);
            if (posCollToken != address(wIchiFarm))
                revert INCORRECT_COLTOKEN(posCollToken);
            bank.takeCollateral(collSize);
            wIchiFarm.burn(collId, collSize);
        }
```

If there was an already existing farming position when calling ````openPositionFarm````, it first takes out the collateral and redeems the underlying tokens by calling ````wIchiFarm.burn````.

```solidity
        // Transfer LP Tokens
        address lpToken = ichiFarm.lpToken(pid);
        IERC20Upgradeable(lpToken).safeTransfer(msg.sender, amount);

        // Transfer Reward Tokens
        (uint256 enIchiPerShare, , ) = ichiFarm.poolInfo(pid);
        uint256 stIchi = (stIchiPerShare * amount).divCeil(1e18);
        uint256 enIchi = (enIchiPerShare * amount) / 1e18;

        if (enIchi > stIchi) {
            ICHI.safeTransfer(msg.sender, enIchi - stIchi);
        }
```
````wIchiFarm.burn```` returns the LP tokens and ICHI farming results to the spell contract.

```solidity
        // 5. Deposit on farming pool, put collateral
        ensureApprove(strategy.vault, address(wIchiFarm));
        uint256 lpAmount = IERC20(strategy.vault).balanceOf(address(this));
        uint256 id = wIchiFarm.mint(farmingPid, lpAmount);
        bank.putCollateral(address(wIchiFarm), id, lpAmount);
```
The LP tokens are wrapped and deposited to the farm again, but the ICHI tokens are still left in the contract.
As written in the specs, spell contracts should not hold assets, and the extra tokens kept in the contract can be utilized by anyone calling the contract. As a result, it will result in a loss of assets and also make the contract vulnerable to back-running attacks.

## Impact

User funds(reward tokens) are lost, can be target to back-running attacks

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L199-L249

## Tool used

Manual Review

## Recommendation

Add a collecting logic for ICHI
```solidity
        // 5. Deposit on farming pool, put collateral
        ensureApprove(strategy.vault, address(wIchiFarm));
        uint256 lpAmount = IERC20(strategy.vault).balanceOf(address(this));
        uint256 id = wIchiFarm.mint(farmingPid, lpAmount);
        bank.putCollateral(address(wIchiFarm), id, lpAmount);

+        doRefund(ICHI);
    }
```
or re-deposit the tokens if the vault is an ICHI pair.
