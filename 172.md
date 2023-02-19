foxb868

high

# Insufficient Balance Check in "Withdraw" Function.

## Summary
In detail.

## Vulnerability Detail
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/vault/SoftVault.sol#L94-L124
Let me dive deeper into the code. The withdraw function first checks if the `shareAmount` parameter is not equal to zero, If it is, the function reverts by calling the `ZERO_AMOUNT()` function from BlueBerryErrors library.

After that, the function calls the `_burn` function to burn the `shareAmount` tokens from the user's balance the `_burn` function is an internal function of the "ERC20Upgradeable" contract, which reduces the total supply of the token and subtracts the `shareAmount` tokens from the balance of the user's account.

The function then gets the balance of the underlying token before calling the redeem function of the `cToken` contract to redeem the `shareAmount` tokens. If the redeem function returns a non-zero value, the function reverts by calling the `REDEEM_FAILED` function from `BlueBerryErrors` library.

After that, the function gets the balance of the underlying token again and calculates the `withdrawAmount` by subtracting the previous balance from the current balance. If the current balance is less than the previous balance, the contract is stuck.

If the withdraw fee is enabled, the function calculates the fee amount based on the `withdrawVaultFee` and `withdrawVaultFeeWindow` parameters from the config contract and transfers the fee to the treasury address, the remaining `withdrawAmount` is then transferred to the user's address.

## Impact
If the underlying token balance is not checked before redeeming `cTokens`, and the balance is insufficient, the `redeem` operation will fail, the contract will get stuck and the users who attempt to withdraw funds will be unable to do so, and their funds will be lost in the contract. 

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/vault/SoftVault.sol#L105
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/vault/SoftVault.sol#L94-L124
## Tool used

Manual Review

## Recommendation
Add a check to the `withdraw` function that verifies whether the underlying token balance is sufficient before redeeming `cTokens`.
If the balance is not enough, the contract should revert, and the user's funds will remain safe in the contract in that way.

If I may put it in code, here is how it will be the implementation.
```solidity
/**
 * @notice Withdraw underlying assets from Compound
 * @param shareAmount Amount of cTokens to redeem
 * @return withdrawAmount Amount of underlying assets withdrawn
 */
function withdraw(uint256 shareAmount)
    external
    override
    nonReentrant
    returns (uint256 withdrawAmount)
{
    if (shareAmount == 0) revert ZERO_AMOUNT();

    _burn(msg.sender, shareAmount);

    uint256 uBalanceBefore = uToken.balanceOf(address(this));
    require(cToken.balanceOf(address(this)) >= shareAmount, "Insufficient cToken balance");
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
In here i have added a `require` statement that checks if the balance of the `cToken` is greater than or equal to the `shareAmount` being redeemed. If the balance is less than the `shareAmount`, the contract will revert, and the user's funds will be safe.
