ctf_sec

medium

# Interest should not be accrued when repay is disabled

## Summary

Interest should not be accrued when repay is disabled

## Vulnerability Detail

In the BllueBerryBank.sol,

we have a function

```solidity
/// @dev Ensure that the interest rate of the given token is accrued.
modifier poke(address token) {
    accrue(token);
    _;
}
```

which calls:

```solidity
   /// @dev Trigger interest accrual for the given bank.
    /// @param token The underlying token to trigger the interest accrual.
    function accrue(address token) public override {
        Bank storage bank = banks[token];
        if (!bank.isListed) revert BANK_NOT_LISTED(token);
        bank.totalDebt = ICErc20(bank.cToken).borrowBalanceCurrent(
            address(this)
        );
    }
```

However, the repay can be disabled

```solidity
    /// @dev Bank repay status allowed or not
    /// @notice Check second-to-last bit of bankStatus
    function isRepayAllowed() public view returns (bool) {
        return (bankStatus & 0x02) > 0;
    }
```

If the repay function is disabled and the interest continue to accure, it is not very fair for the borrower because the they have no choice to watch the debt keeps growing until they get liquidated which very heavily disincentize them to borrow and generate yield for lenders. 

## Impact

If the repay function is disabled and the interest continue to accure, it is not very fair for the borrower because the they have no choice to watch the debt keeps growing until they get liquidated which very heavily disincentize them to borrow and generate yield for lenders. 

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L84-L90

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L248-L258

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L736-L755

## Tool used

Manual Review

## Recommendation

We recommend the protocol not accuring interest when the repay is disabled.
