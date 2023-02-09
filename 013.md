0x52

medium

# BlueBerryBank#doCutDepositFee is problematic for ERC20 tokens that don't support zero transfers

## Summary

BlueBerryBank#doCutDepositFee attempts to transfer funds even if there isn't a deposit fee. If the underlying ERC20 doens't support transfers with zero value then the call will always revert when the deposit fee is zero.

## Vulnerability Detail

    function doCutDepositFee(address token, uint256 amount)
        internal
        returns (uint256)
    {
        if (config.treasury() == address(0)) revert NO_TREASURY_SET();
        uint256 fee = (amount * config.depositFee()) / DENOMINATOR;
        IERC20Upgradeable(token).safeTransfer(config.treasury(), fee);
        return amount - fee;
    }

BlueBerryBank#doCutDepositFee always calls safeTransfer even when the amount to send could be zero because there isn't any deposit fee. For ERC20 tokens that don't support zero value transfers, this will always revert which breaks support for them.

## Impact

Having no deposit fee will break support for ERC20 tokens that don't support zero transfers

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L892-L900

## Tool used

Manual Review

## Recommendation

Only transfer if the amount to transfer is not zero:

    function doCutDepositFee(address token, uint256 amount)
        internal
        returns (uint256)
    {
        if (config.treasury() == address(0)) revert NO_TREASURY_SET();
        uint256 fee = (amount * config.depositFee()) / DENOMINATOR;
    -   IERC20Upgradeable(token).safeTransfer(config.treasury(), fee);
    +   if (fee != 0) IERC20Upgradeable(token).safeTransfer(config.treasury(), fee);
        return amount - fee;
    }
