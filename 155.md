obront

high

# totalLend isn't updated on liquidation, leading to permanently inflated value

## Summary

`bank.totalLend` tracks the total amount that has been lent of a given token, but it does not account for tokens that are withdrawn when a position is liquidated. As a result, the value will become overstated, leading to inaccurate data on the pool.

## Vulnerability Detail

When a user lends a token to the Compound fork, the bank for that token increases its `totalLend` parameter:
```solidity
bank.totalLend += amount;
```
Similarly, this value is decreased when the amount is withdrawn.

In the event that a position is liquidated, the `underlyingAmount` and `underlyingVaultShare` for the user are decreased based on the amount that will be transferred to the liquidator.
```solidity
uint256 liqSize = (pos.collateralSize * share) / oldShare;
uint256 uTokenSize = (pos.underlyingAmount * share) / oldShare;
uint256 uVaultShare = (pos.underlyingVaultShare * share) / oldShare;

pos.collateralSize -= liqSize;
pos.underlyingAmount -= uTokenSize;
pos.underlyingVaultShare -= uVaultShare;
```

However, the liquidator doesn't receive those shares "inside the system". Instead, they receive the softVault tokens that can be claimed directly for the underlying asset by calling `withdraw()`, which simply redeems the underlying tokens from the Compound fork and sends them to the user.
```solidity
function withdraw(uint256 shareAmount)
    external
    override
    nonReentrant
    returns (uint256 withdrawAmount)
{
    if (shareAmount == 0) revert ZERO_AMOUNT();

    _burn(msg.sender, shareAmount);

    uint256 uBalanceBefore = uToken.balanceOf(address(this));
    if (cToken.redeem(shareAmount) != 0) revert REDEEM_FAILED(shareAmount);
    uint256 uBalanceAfter = uToken.balanceOf(address(this));

    withdrawAmount = uBalanceAfter - uBalanceBefore;
    // Cut withdraw fee if it is in withdrawVaultFee Window (2 months)
    if (
        block.timestamp <
        config.withdrawVaultFeeWindowStartTime() +
            config.withdrawVaultFeeWindow()
    ) {
        uint256 fee = (withdrawAmount * config.withdrawVaultFee()) /
            DENOMINATOR;
        uToken.safeTransfer(config.treasury(), fee);
        withdrawAmount -= fee;
    }
    uToken.safeTransfer(msg.sender, withdrawAmount);

    emit Withdrawn(msg.sender, withdrawAmount, shareAmount);
}
```

Nowhere in this process is `bank.totalLend` updated. As a result, each time there is a liquidation of size X, `bank.totalLend` will move X higher relative to the correct value. Slowly, over time, this value will begin to dramatically misrepresent the accurate amount that has been lent.

While there is no material exploit based on this inaccuracy at the moment, this is a core piece of data in the protocol, and it's inaccuracy could lead to major issues down the road. 

Furthermore, it will impact immediate user behavior, as the Blueberry devs have explained "we use that [value] to help us display TVL with subgraph", which will deceive and confuse users.

## Impact

A core metric of the protocol will be permanently inaccurate, giving users incorrect data to make their assessments on and potentially causing more severe issues down the road.

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L511-L572

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/vault/SoftVault.sol#L94-L123

## Tool used

Manual Review

## Recommendation

For the best accuracy, updating `bank.totalLend` should happen from the `withdraw()` function in `SoftVault.sol` instead of from the core `BlueberryBank.sol` contract.

Alternatively, you could add an update to `bank.totalLend` in the `liquidate()` function, which might temporarily underrepresent the total lent before the liquidator withdrew the funds, but would end up being accurate over the long run.