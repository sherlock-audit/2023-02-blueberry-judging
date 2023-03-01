chaduke

medium

# SoftVault.deposit() will always REVERT due to a bug

## Summary
``SoftVault.deposit()`` will always REVERT due to a bug.

## Vulnerability Detail
The following condition in L79  ``SoftVault.deposit()`` seems to have an error. It could be just a typo. 

[https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/vault/SoftVault.sol#L67-L87](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/vault/SoftVault.sol#L67-L87)

The logic should be: when it returns zero shares, we need to revert. Here is the correction:

```javascript
if (cToken.mint(uBalanceAfter - uBalanceBefore) == 0)
            revert LEND_FAILED(amount);"
```

## Impact
``SoftVault.deposit()`` will always REVERT due to a bug.


## Code Snippet
see above

## Tool used
VSCode

Manual Review

## Recommendation
Correction as follows:
```diff
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
-        if (cToken.mint(uBalanceAfter - uBalanceBefore) != 0)
+       if (cToken.mint(uBalanceAfter - uBalanceBefore) == 0)
            revert LEND_FAILED(amount);
        uint256 cBalanceAfter = cToken.balanceOf(address(this));

        shareAmount = cBalanceAfter - cBalanceBefore;
        _mint(msg.sender, shareAmount);

        emit Deposited(msg.sender, amount, shareAmount);
    }
```