chaduke

medium

# Some funds might be stuck in the bank contract forever, and nobody can withdraw them

## Summary
Some funds might be stuck in the bank contract forever, and nobody can withdraw them. 


## Vulnerability Detail
We show below how some funds might be stuck in the bank contract forever.

1) When ``BlueBerryBank.withdrawLend()`` is called, it will withdraw ``wAmount`` of the underlying tokens from the vault that corresponds to  ``shareAmount`` of vault shares. 

[https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L669-L704](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L669-L704)

2) However, when ``wAmount > pos.underlyingAmount``, only ``pos.underlyingAmount`` of underlying tokens will be sent back to the user (minus the fee), the remaining ``wAmount - pos.underlyingAmount`` underlying tokens will be stuck in the bank contract.

3) There are no functions that will  allow an owner/admin to withdraw these locked funds, they are lost. 

## Impact
Some funds might be locked in the bank contract and thus lost forever. 



## Code Snippet
See above

## Tool used
Remix, VScode

Manual Review

## Recommendation
The ``withdrawLend()`` function should send all these tokens back to the user. 
 