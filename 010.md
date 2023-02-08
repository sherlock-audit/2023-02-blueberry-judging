koxuan

high

# First depositor can manipulate share price of SoftVault

## Summary
First depositor can mint 1 share and send in large amount of uToken into `SoftVault`. The price of 1 share will be inflated. The next depositor who deposit will lose out due to rounding error. 


## Vulnerability Detail

Adversary will call `deposit` with amount 1 to mint 1 share. He then transfer large amount of uToken over 1000. The price of 1 share is now 1001. The next depositor deposits 2000 and receives 1 share due to rounding error. The pool has 3001 asset with 2 shares. Adversary can withdraw his 1 share and receive 1500 asset, gaining 500 asset. 

```solidity
    function deposit(uint256 amount)
        external
        override
        nonReentrant
        returns (uint256 shareAmount)
    {
        if (amount == 0) revert ZERO_AMOUNT();
        uint256 uBalanceBefore = uToken.balanceOf(address(this));
        uToken.safeTransferFrom(msg.sender, address(this), amount);
        uint256 uBalanceAfter = uToken.balanceOf(address(this));

        uint256 cBalanceBefore = cToken.balanceOf(address(this));
        if (cToken.mint(uBalanceAfter - uBalanceBefore) != 0)
            revert LEND_FAILED(amount);
        uint256 cBalanceAfter = cToken.balanceOf(address(this));

        shareAmount = cBalanceAfter - cBalanceBefore;
        _mint(msg.sender, shareAmount);

        emit Deposited(msg.sender, amount, shareAmount);
    }

```
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
}
```

## Impact
First depositor can manipulate share price to gain from future depositors.

## Code Snippet
[SoftVault.sol#L67-L87](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/vault/SoftVault.sol#L67-L87)
[SoftVault.sol#L94-L124](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/vault/SoftVault.sol#L94-L124)
## Tool used

Manual Review

## Recommendation

Recommend requiring minimum amount of shares for first depositor and burning some shares of first depositor so that price of share will be more resistant.
