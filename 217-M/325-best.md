tsvetanovv

medium

# Possibility of borrowing more tokens than are in reserve

## Summary
In [BlueBerryBank.sol](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L709) we have `borrow()` function. This function allow to borrow tokens from given bank. This function has no limit how many tokens I can borrow.

## Vulnerability Detail
Тhe borrower could borrow as many assets from the Bank as they wish or all the assets from the Bank.

## Impact
Possibility of borrowing more tokens than the bank has.

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L709
```solidity
function borrow(address token, uint256 amount)
        external
        override
        inExec
        poke(token)
        onlyWhitelistedToken(token)
    {
        if (!isBorrowAllowed()) revert BORROW_NOT_ALLOWED();
        Bank storage bank = banks[token];
        Position storage pos = positions[POSITION_ID];
        uint256 totalShare = bank.totalShare;
        uint256 totalDebt = bank.totalDebt;
        uint256 share = totalShare == 0
            ? amount
            : (amount * totalShare).divCeil(totalDebt);
        bank.totalShare += share;
        uint256 newShare = pos.debtShareOf[token] + share;
        pos.debtShareOf[token] = newShare;
        if (newShare > 0) {
            pos.debtMap |= (1 << uint256(bank.index));
        }
        IERC20Upgradeable(token).safeTransfer(
            msg.sender,
            doBorrow(token, amount)
        );
        emit Borrow(POSITION_ID, msg.sender, token, amount, share);
    }
```

## Tool used

Manual Review

## Recommendation

Make some checks so that a user cannot withdraw more tokens than the bank has or withdraw the entire bank reserve.