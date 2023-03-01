y1cunhui

high

# `WIchiFarm.mint` may fail if the lpToken spendAllowance also decrease type(uint256).max

## Summary
The `WIchiFarm.mint` may permanently fail for lpToken that also decrease the amount of allowance in `_spendAllowance`.
## Vulnerability Detail

In the code snippet shown below, the WIchiFarm checked if the allowance is `type(uint256).max`; if not, it calls `SafeERC20Upgradeable.safeApprove` to approve the max amount. In the comment, it says "We only need to do this once per pool, as LP token's allowance won't decrease if it's -1.". However, this behaviour is not declared in ERC20 Standard. If the `lpToken` implementation also spend allownance of `type(uint256).max`, the `safeApprove` would fail because it requires the previous allowance to be zero:
```solidity
// SafeERC20Upgradeable.sol
    /**
     * @dev Deprecated. This function has issues similar to the ones found in
     * {IERC20-approve}, and its usage is discouraged.
     *
     * Whenever possible, use {safeIncreaseAllowance} and
     * {safeDecreaseAllowance} instead.
     */
    function safeApprove(IERC20Upgradeable token, address spender, uint256 value) internal {
        // safeApprove should only be called when setting an initial allowance,
        // or when resetting it to zero. To increase and decrease it, use
        // 'safeIncreaseAllowance' and 'safeDecreaseAllowance'
        require(
            (value == 0) || (token.allowance(address(this), spender) == 0),
            "SafeERC20: approve from non-zero to non-zero allowance"
        );
        _callOptionalReturn(token, abi.encodeWithSelector(token.approve.selector, spender, value));
    }
```

## Impact
This will cause the `mint` function for this token PERMANENTLY unavailable.

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/wrapper/WIchiFarm.sol#L93-L104
## Tool used

Manual Review

## Recommendation
There are 2 choices:
1. Change the `safeApprove` into `safeIncreaseAllowance`, since `safeApprove`is deprecated;
2. Change the if condition to 
```solidity
       if (
            IERC20Upgradeable(lpToken).allowance(
                address(this),
                address(ichiFarm)
            ) == 0
        ) {
            ...
        }
```
to make the `safeApprove` always available(use it when setting an initial allowance).