Ch_301

medium

# The SPELL contract is designed to receive ETH only from WETH but not implement a function for Withdrawal

## Summary
The SPELL contract instances can [receive()](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/BasicSpell.sol#L141-L143) ETH but no one can withdraw it, Only the WETH contract can send ETH 

## Vulnerability Detail
SPELL doesn't have a function to rescue the native tokens sent by the WETH contract.

## Impact
the protocol can't withdraw native tokens that are sent by the WETH contract.

## Code Snippet
```solidity
    receive() external payable {
        if (msg.sender != weth) revert NOT_FROM_WETH(msg.sender);
    }
```

## Tool used

Manual Review

## Recommendation
Add a `rescueETH()` function
