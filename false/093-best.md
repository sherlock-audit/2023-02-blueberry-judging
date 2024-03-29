0xbrett8571

medium

# logic error in the design of the lending process.

## Summary
The lend function does not check for the balance of the user that wants to lend, to ensure they have sufficient funds to lend the amount of tokens.

## Vulnerability Detail
The contract provides functions for creating a new position, executing actions, and liquidating a position.

But the `lend` function doesn't check the balance of the "user" to ensure that they have sufficient funds/collateral to lend the amount of tokens. 

This can allow someone that understands the flaw to lend more tokens than they have in their account, leading to a position where the contract is unable to transfer the tokens back to the user.

## Impact
`lend` function does not check for the balance of the user, to ensure they have sufficient funds to lend the amount of tokens. As a result, users can lend more tokens than they actually own, leading to an inconsistent state.

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L620-L625
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L626-L635

## Tool used

Manual Review

## Recommendation
The `lend` function should include a check to confirm that the user who wants to lend has enough balance to complete the lend operation.

Add this line of code before the deposit to the bank.
```solidity
+ require(IERC20(token).allowance(msg.sender, address(this)) >= amount, "Insufficient funds");
```