clems4ever

medium

# Safety net on computing total debt after borrowing

## Summary

bank.totalDebt is updated when accrual is computed, borrowing is done or a repayment is made. In the case of accrual and repayment the actual value of the debt is coming from the cToken contract ensuring that the value is actual. When borrowing though, the value `amountCall` is added to the debt with no guarantee whether this amount has actually been borrowed.

I suggest that you align the computation of `totalDebt` with the other instances.

## Vulnerability Detail

The value bank.totalDebt is updated with the value from the cToken in both cases below:

```solidity
function doRepay(address token, uint256 amountCall)
        internal
        returns (uint256 repaidAmount)
    {
        Bank storage bank = banks[token]; // assume the input is already sanity checked.
        IERC20Upgradeable(token).approve(bank.cToken, amountCall);
        if (ICErc20(bank.cToken).repayBorrow(amountCall) != 0)
            revert REPAY_FAILED(amountCall);
        uint256 newDebt = ICErc20(bank.cToken).borrowBalanceStored( <==========================================
            address(this)
        );
        repaidAmount = bank.totalDebt - newDebt;
        bank.totalDebt = newDebt;
    }
```

```solidity
function accrue(address token) public override {
        Bank storage bank = banks[token];
        if (!bank.isListed) revert BANK_NOT_LISTED(token);
        bank.totalDebt = ICErc20(bank.cToken).borrowBalanceCurrent( <========================================
            address(this)
        );
    }
```

But in the case of a borrow the value is not coming from the cToken after the actual borrow as you can see in the snippet below

```solidity
function doBorrow(address token, uint256 amountCall)
        internal
        returns (uint256 borrowAmount)
    {
        Bank storage bank = banks[token]; // assume the input is already sanity checked.

        IERC20Upgradeable uToken = IERC20Upgradeable(token);
        uint256 uBalanceBefore = uToken.balanceOf(address(this));
        if (ICErc20(bank.cToken).borrow(amountCall) != 0)
            revert BORROW_FAILED(amountCall);
        uint256 uBalanceAfter = uToken.balanceOf(address(this));

        borrowAmount = uBalanceAfter - uBalanceBefore;
        bank.totalDebt += amountCall; <====================================================
    }
```

This can be an issue in two cases:

1. The amount actually borrowed is less than amountCall, then the totalDebt will be recorded as bigger that what it actually is until the next accrual happens. In one case, the position might end up liquidatable and the operation execution will revert because of https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L607.
or it can be not reverting and recording the wrong debt temporarily

2. The amount actually borrowed is bigger than amountCall (for instance because some fees might be accounted in the interests when entering the loan) and in that case the amount recorded into totalDebt might be less than what it is supposed to be until the next operation.

## Impact

All view functions exposing the debt and the risk will not be accurate.

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L868

## Tool used

Manual Review

## Recommendation

When borrowing, use the actual debt as seen by the cToken.

```solidity
function doBorrow(address token, uint256 amountCall)
        internal
        returns (uint256 borrowAmount)
    {
        Bank storage bank = banks[token]; // assume the input is already sanity checked.

        IERC20Upgradeable uToken = IERC20Upgradeable(token);
        uint256 uBalanceBefore = uToken.balanceOf(address(this));
        if (ICErc20(bank.cToken).borrow(amountCall) != 0)
            revert BORROW_FAILED(amountCall);
        uint256 uBalanceAfter = uToken.balanceOf(address(this));

        borrowAmount = uBalanceAfter - uBalanceBefore;
        uint256 newDebt = ICErc20(bank.cToken).borrowBalanceStored(
            address(this)
        );
        bank.totalDebt += newDebt;
    }
```