0x52

high

# WIchiFarm will break after second deposit of LP

## Summary

WIchiFarm.sol makes the incorrect assumption that IchiVaultLP doesn't reduce allowance when using the transferFrom if allowance is set to type(uint256).max. Looking at a currently deployed [IchiVault](https://etherscan.io/token/0x683f081dbc729dbd34abac708fa0b390d49f1c39#code#L2281) this assumption is not true. On the second deposit for the LP token, the call will always revert at the safe approve call.

## Vulnerability Detail

[IchiVault](https://etherscan.io/token/0x683f081dbc729dbd34abac708fa0b390d49f1c39#code#L2281)

      function transferFrom(address sender, address recipient, uint256 amount) public virtual override returns (bool) {
          _transfer(sender, recipient, amount);
          _approve(sender, _msgSender(), _allowances[sender][_msgSender()].sub(amount, "ERC20: transfer amount exceeds allowance"));
          return true;
      }

The above lines show the trasnferFrom call which reduces the allowance of the spender regardless of whether the spender is approved for type(uint256).max or not. 

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

As a result after the first deposit the allowance will be less than type(uint256).max. When there is a second deposit, the reduced allowance will trigger a safeApprove call.

    function safeApprove(
        IERC20Upgradeable token,
        address spender,
        uint256 value
    ) internal {
        // safeApprove should only be called when setting an initial allowance,
        // or when resetting it to zero. To increase and decrease it, use
        // 'safeIncreaseAllowance' and 'safeDecreaseAllowance'
        require(
            (value == 0) || (token.allowance(address(this), spender) == 0),
            "SafeERC20: approve from non-zero to non-zero allowance"
        );
        _callOptionalReturn(token, abi.encodeWithSelector(token.approve.selector, spender, value));
    }

safeApprove requires that either the input is zero or the current allowance is zero. Since neither is true the call will revert. The result of this is that WIchiFarm is effectively broken after the first deposit.

## Impact

WIchiFarm is broken and won't be able to process deposits after the first.

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/wrapper/WIchiFarm.sol#L38

## Tool used

Manual Review

## Recommendation

Only approve is current allowance isn't enough for call. Optionally add zero approval before the approve. Realistically it's impossible to use the entire type(uint256).max, but to cover edge cases you may want to add it.

        if (
            IERC20Upgradeable(lpToken).allowance(
                address(this),
                address(ichiFarm)
    -       ) != type(uint256).max
    +       ) < amount
        ) {

    +       IERC20Upgradeable(lpToken).safeApprove(
    +           address(ichiFarm),
    +           0
            );
            // We only need to do this once per pool, as LP token's allowance won't decrease if it's -1.
            IERC20Upgradeable(lpToken).safeApprove(
                address(ichiFarm),
                type(uint256).max
            );
        }