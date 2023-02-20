tsvetanovv

medium

# No storage gap for upgradable contracts might lead to storage slot collision

## Summary
For upgradeable contracts, there must be storage gap to “allow developers to freely add new state variables in the future without compromising the storage compatibility with existing deployments”. 
Otherwise, it may be very difficult to write new implementation code. Without storage gap, the variable in the contract contract might be overwritten by the upgraded contract if new variables are added. 
This could have unintended and very serious consequences to the child contracts.
Only [ERC1155NaiveReceiver.sol](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/utils/ERC1155NaiveReceiver.sol#L8) have storage gap.

## Vulnerability Detail

The storage gap is essential for upgradeable contract because “It allows us to freely add new state variables in the future without compromising the storage compatibility with existing deployments”. 

## Impact
Without storage gap, the variable in the contract contract might be overwritten by the upgraded contract if new variables are added.  This could have unintended and very serious consequences to the child contracts.

## Code Snippet


## Tool used
Manual Review

## Recommendation
Consider defining an appropriate storage gap in each upgradeable parent contract at the end of all the storage variable definitions as follows:

```solidity
	uint256[50] __gap; // gap to reserve storage in the contract for future variable additions
```