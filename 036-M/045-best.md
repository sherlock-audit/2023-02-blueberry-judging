chaduke

high

# A borrower might drain the vault by calling borrow() repeatedly with small borrow amount each time.

## Summary
A borrower might drain the vault by calling ``borrow()`` repeatedly with small borrow amount that converts to a zero debt share each time.  

## Vulnerability Detail
This is possible because ``borrow()`` does not check whether the number of shares borrowed is equal to zero or not. Therefore, an attacker can take advantage of the rounding error and borrow funds for free. We show how a borrowed can drain the vault by calling borrow() repeatedly:

1) Suppose the for a particular token X, the total bank debt is 1000,000 and the total debt share is 100,000. That is each debt share has a 10 debt. 

2) A malicious borrower Bob can call ``borrow()`` (via SPELL) and borrow 9 each time, which will convert to ``9*100,000/1000,000 = 0`` debt shares.

[https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L709-L735](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L709-L735)

3) As a result, the borrower can steal 9 X tokens each time the ``borrow()`` is called without increasing his debt shares. Bob can call this repeatedly in one transaction ``Steal()`` (within the gas limit) to borrow many tokens of X without increasing any debt shares. 

4) Call ``Steal()`` many times, Bob will be able to drain the vault. 
 

## Impact
A malicious borrower can drain the the vault by calling ``borrow()`` repeatedly.

## Code Snippet
See above

## Tool used
VScode
Manual Review

## Recommendation
Borrow should revert when ``newShare == 0``.
```diff
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
+       if(share == 0) revert BorrowZeroShare();

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
