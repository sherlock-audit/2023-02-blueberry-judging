chaduke

medium

# A borrower might receive ZERO underlying tokens after burning his shares in ``pos.underlyingVaultShare``.

## Summary
A borrower might receive ZERO underlying tokens after burning his shares in ``pos.underlyingVaultShare``.

## Vulnerability Detail
We show how a borrower might receive ZERO underlying tokens after burning his shares in ``pos.underlyingVaultShare`` or receive much less tokens than these shares are worth. 

1) Suppose Bob has a position: ``pos.underlyingVaultShare = 100``, ``pos.underlyingAmount = 5000``, and suppose that now each vault share is worth 100 underlying tokens. 

2) Bob will call ``BlueBerryBank.withdrawLend`` (via execute and then via SPELL) to withdraw 50 shares. As a result, he will get (5000-fee) underlying tokens. and then ``pos.underlyingAmount = 0``, ``pos.underlyingVaultShare = 50`` (L697-L699)

[https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L669-L704](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L669-L704)

3) However, if Bob tries to withdraw another 50 shares, although all these 50 shares will be burned and the bank will thus get 5000 underlying tokens,  ZERO underlying tokens will be sent to Bob because ``wAmount`` will be set to zero due to ``pos.underlyingAmount = 0``. If ``pos.underlyingAmount`` has a small value, only that small amount will be sent back to Bob.
 
```solidity
  wAmount = wAmount > pos.underlyingAmount
            ? pos.underlyingAmount
            : wAmount;

        pos.underlyingVaultShare -= shareAmount;
        pos.underlyingAmount -= wAmount;
        bank.totalLend -= wAmount;

        wAmount = doCutWithdrawFee(token, wAmount);

        IERC20Upgradeable(token).safeTransfer(msg.sender, wAmount);```
```

## Impact
A borrower might receive ZERO collateral tokens after burning his shares in ``pos.underlyingVaultShare``, or he might receive much less underlying tokens than his shares deserve.

## Code Snippet
See above

## Tool used
Remix

Manual Review

## Recommendation
The withdrawn underlying tokens should be all sent to the user because  he earns it. 
