chaduke

high

# A borrower can borrow tokens without any collateral at all..

## Summary
A borrower can borrow tokens without any collateral at all. The main issue is that the risk evaluation will consider the risk is ZERO when there is ZERO amount of underlying tokens. 

## Vulnerability Detail
Let's consider how a borrower Alice can borrow tokens without any collateral at all:

1) Suppose Alice calls ``borrow()`` through SPELL to borrow 1,000,000 X tokens; 

2) After the completion of the execution of ``borrow()``, the ``isLiquidatable()`` will be called to ensure the risk is acceptable, that is, it is not greater than a threshold. 

```javascript
   if (isLiquidatable(positionId)) revert INSUFFICIENT_COLLATERAL();

 function isLiquidatable(uint256 positionId)
        public
        view
        returns (bool liquidatable)
    {
        Position storage pos = positions[positionId];
        uint256 risk = getPositionRisk(positionId);
        liquidatable = risk >= oracle.getLiqThreshold(pos.underlyingToken);
    }
```
3)``getPositionRisk(positionId)`` is called to evaluate the risk of ``positionId``.  The problem lies in L490. When this is  zero amount of underlying tokens, ``cv = 0``, then it will return risk = 0, even in the case of zero collateral. 

[https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L477-L495](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L477-L495)

4) As a result, Alice can borrow 1,000,000 tokens for free without any collateral. 

## Impact
A borrower can borrow tokens without any collateral at all, and thus can easily drain the vault. 

## Code Snippet
[https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L709-L735](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L709-L735)

## Tool used
VScode

Manual Review

## Recommendation
Make sure the risk evaluation will be more robust, for example,  consider the ratio of debt vs the sum of underlying token value and collateral value.
