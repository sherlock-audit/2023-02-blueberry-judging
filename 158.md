obront

high

# Users who deposit extra funds into their Ichi farming positions will lose all their ICHI rewards

## Summary

When a user deposits extra funds into their Ichi farming position using `openPositionFarm()`, the old farming position will be closed down and a new one will be opened. Part of this process is that their ICHI rewards will be sent to the `IchiVaultSpell.sol` contract, but they will not be distributed. They will sit in the contract until the next user (or MEV bot) calls `closePositionFarm()`, at which point they will be stolen by that user.

## Vulnerability Detail

When Ichi farming positions are opened via the `IchiVaultSpell.sol` contract, `openPositionFarm()` is called. It goes through the usual deposit function, but rather than staking the LP tokens directly, it calls `wIchiFarm.mint()`. This function deposits the token into the `ichiFarm`, encodes the deposit as an ERC1155, and sends that token back to the Spell:
```solidity
function mint(uint256 pid, uint256 amount)
    external
    nonReentrant
    returns (uint256)
{
    address lpToken = ichiFarm.lpToken(pid);
    IERC20Upgradeable(lpToken).safeTransferFrom(
        msg.sender,
        address(this),
        amount
    );
    if (
        IERC20Upgradeable(lpToken).allowance(
            address(this),
            address(ichiFarm)
        ) != type(uint256).max
    ) {
        // We only need to do this once per pool, as LP token's allowance won't decrease if it's -1.
        IERC20Upgradeable(lpToken).safeApprove(
            address(ichiFarm),
            type(uint256).max
        );
    }
    ichiFarm.deposit(pid, amount, address(this));
    // @ok if accIchiPerShare is always changing, so how does this work?
    // it's basically just saving the accIchiPerShare at staking time, so when you unstake, it can calculate the difference
    // really fucking smart actually
    (uint256 ichiPerShare, , ) = ichiFarm.poolInfo(pid);
    uint256 id = encodeId(pid, ichiPerShare);
    _mint(msg.sender, id, amount, "");
    return id;
}
```
The resulting ERC1155 is posted as collateral in the Blueberry Bank.

If the user decides to add more funds to this position, they simply call `openPositionFarm()` again. The function has logic to check if there is already existing collateral of this LP token in the Blueberry Bank. If there is, it removes the collateral and calls `wIchiFarm.burn()` (which harvests the Ichi rewards and withdraws the LP tokens) before repeating the deposit process.
```solidity
function burn(uint256 id, uint256 amount)
    external
    nonReentrant
    returns (uint256)
{
    if (amount == type(uint256).max) {
        amount = balanceOf(msg.sender, id);
    }
    (uint256 pid, uint256 stIchiPerShare) = decodeId(id);
    _burn(msg.sender, id, amount);

    uint256 ichiRewards = ichiFarm.pendingIchi(pid, address(this));
    ichiFarm.harvest(pid, address(this));
    ichiFarm.withdraw(pid, amount, address(this));

    // Convert Legacy ICHI to ICHI v2
    if (ichiRewards > 0) {
        ICHIv1.safeApprove(address(ICHI), ichiRewards);
        ICHI.convertToV2(ichiRewards);
    }

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
    return pid;
}
```
However, this deposit process has no logic for distributing the ICHI rewards. Therefore, these rewards will remain sitting in the `IchiVaultSpell.sol` contract and will not reach the user.

For an example of how this is handled properly, we can look at the opposite function, `closePositionFarm()`. In this case, the same `wIchiFarm.burn()` function is called. But in this case, it's followed up with an explicit call to withdraw the ICHI from the contract to the user.
```solidity
doRefund(ICHI);
```
This `doRefund()` function refunds the contract's full balance of ICHI to the `msg.sender`, so the result is that the next user to call `closePositionFarm()` will steal the ICHI tokens from the original user who added to their farming position.

## Impact

Users who farm their Ichi LP tokens for ICHI rewards can permanently lose their rewards.

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L199-L249

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/wrapper/WIchiFarm.sol#L116-L150

Here is a link to the `harvest()` function  on the IchiFarmV2.sol contract, which is called by `wIchiFarm.sol` and contains the logic for distributing ICHI rewards: https://github.com/ichifarm/ichi-farming/blob/206c44b790fbb2a1e3a655685eb3ab8d793c9f00/contracts/ichiFarmV2.sol#L238-L257

## Tool used

Manual Review

## Recommendation

In the `openPositionFarm()` function, in the section that deals with withdrawing existing collateral, add a line that claims the ICHI rewards for the calling user.

```diff
if (collSize > 0) {
    (uint256 decodedPid, ) = wIchiFarm.decodeId(collId);
    if (farmingPid != decodedPid) revert INCORRECT_PID(farmingPid);
    if (posCollToken != address(wIchiFarm))
        revert INCORRECT_COLTOKEN(posCollToken);
    bank.takeCollateral(collSize);
    wIchiFarm.burn(collId, collSize);
+   doRefund(ICHI);
}
```