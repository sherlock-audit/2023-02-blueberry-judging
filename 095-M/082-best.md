rvierdiiev

medium

# BasicSpell.doCutRewardsFee uses depositFee instead of withdraw fee

## Summary
BasicSpell.doCutRewardsFee uses depositFee instead of withdraw fee
## Vulnerability Detail
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/BasicSpell.sol#L65-L79
```solidity
    function doCutRewardsFee(address token) internal {
        if (bank.config().treasury() == address(0)) revert NO_TREASURY_SET();


        uint256 balance = IERC20Upgradeable(token).balanceOf(address(this));
        if (balance > 0) {
            uint256 fee = (balance * bank.config().depositFee()) / DENOMINATOR;
            IERC20Upgradeable(token).safeTransfer(
                bank.config().treasury(),
                fee
            );


            balance -= fee;
            IERC20Upgradeable(token).safeTransfer(bank.EXECUTOR(), balance);
        }
    }
```
This function is called in order to get fee from ICHI rewards, collected by farming.
But currently it takes `bank.config().depositFee()` instead of `bank.config().withdrawFee()`.
## Impact
Wrong fee amount is taken.
## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/BasicSpell.sol#L65-L79
## Tool used

Manual Review

## Recommendation
Take withdraw fee from rewards.