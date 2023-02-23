foxb868

high

# Unprotected User Funds Loss If a user calls the deposit or withdraw function the transfer of tokens will fails

## Summary
The contract doesn't check the return value of `uToken.safeTransferFrom` and `uToken.safeTransfer`, If these functions fail, the contract will continue executing without the transferred tokens, leading to the loss of user funds. The contract should revert if the transfer fails.

## Vulnerability Detail
Unprotected check for the return value of `uToken.safeTransferFrom` and `uToken.safeTransfer` functions, these functions are used to transfer tokens from the sender's address to the contract and from the contract to the recipient's address, respectively. If these functions fail, the contract will continue executing without the transferred tokens.

In the deposit function.
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/vault/SoftVault.sol#L75

in the withdraw function.
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/vault/SoftVault.sol#L120

In both cases, it doesn't check the return value of the `safeTransferFrom` and `safeTransfer` functions. If the transfer fails, the contract will continue executing without the transferred tokens, leading to the loss of user funds.

For instance, in the `deposit` function, if the transfer of tokens from the sender's address to the contract fails, the contract will still try to deposit the tokens on the Compound, which will result in the loss of user funds, similarly, in the `withdraw` function, if the transfer of tokens from the contract to the recipient's address fails, the user will not receive their funds, even though the share token is burned.

## Impact
If a user calls the deposit or withdraw function and the transfer of tokens fails, the user's funds will not be credited or debited, respectively,

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/vault/SoftVault.sol#L75
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/vault/SoftVault.sol#L120

## Tool used

Manual Review

## Recommendation
It should revert if the transfer fails, this can be done by checking the return value of the `safeTransferFrom` and `safeTransfer` functions and reverting if it's false. 
I will give and an example of the `deposit` & `withdraw` functions that can be modified.

In deposit function.
```solidity
function deposit(uint256 amount)
    external
    override
    nonReentrant
    returns (uint256 shareAmount)
{
    if (amount == 0) revert ZERO_AMOUNT();
    uint256 uBalanceBefore = uToken.balanceOf(address(this));
    require(uToken.safeTransferFrom(msg.sender, address(this), amount), "transfer from failed");
    uint256 uBalanceAfter = uToken.balanceOf(address(this));

    uint256 cBalanceBefore = cToken.balanceOf(address(this));
    require(cToken.mint(uBalanceAfter - uBalanceBefore) == 0, "mint failed");
    uint256 cBalanceAfter = cToken.balanceOf(address(this));

    shareAmount = cBalanceAfter - cBalanceBefore;
    _mint(msg.sender, shareAmount);

    emit Deposited(msg.sender, amount, shareAmount);
}
```

In withdraw function.
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
    require(cToken.redeem(shareAmount) == 0, "redeem failed");
    uint256 uBalanceAfter = uToken.balanceOf(address(this));

    withdrawAmount = uBalanceAfter - uBalanceBefore;
    if (
        block.timestamp <
        config.withdrawVaultFeeWindowStartTime() +
            config.withdrawVaultFeeWindow()
    ) {
        uint256 fee = (withdrawAmount * config.withdrawVaultFee()) /
            DENOMINATOR;
        require(uToken.safeTransfer(config.treasury(), fee), "fee transfer failed");
        withdrawAmount -= fee;
    }
    require(uToken.safeTransfer(msg.sender, withdrawAmount), "transfer to user failed");

    emit Withdrawn(msg.sender, withdrawAmount, shareAmount);
  }
}
```