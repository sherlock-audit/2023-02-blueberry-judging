0xbrett8571

high

# `onlyEOAEx` and `onlyWhitelistedToken` modifiers do not include a reentrancy lock.

## Summary
`onlyEOAEx` and `onlyWhitelistedToken` modifiers, do not include a reentrancy lock, leaving the contract vulnerable to reentrancy attacks in those cases.

## Vulnerability Detail
In "onlyEOAEx" and "onlyWhitelistedToken" modifiers, it does not include reentrancy lock, an attacker can heartlessly exploit this vulnerability by any chance repeatedly calling one of these functions before it has completed executing, and it can lead to unintended consequences.
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L52-L65

For example, let's say the `BlueBerryBank` contract has a function that transfers assets/funds from one account to another. 

## Impact
`onlyEOAEx` and `onlyWhitelistedToken` modifiers do not include a reentrancy lock, which in this case leaves the contract vulnerable to reentrancy attacks

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L52-L65

## Tool used

Manual Review

## Recommendation
I will recommend using a reentrancy lock. the lock can be implemented as a simple `boolean` flag that is set to true before executing the main logic of the function and set back to false after the function has been completed.

For example, the "onlyEOAEx" and "onlyWhitelistedToken" modifiers can be updated to include the reentrancy lock.

```solidity
modifier onlyEOAEx() {
+   require(_GENERAL_LOCK == _NOT_ENTERED, "Reentrancy attack detected");
    _GENERAL_LOCK = _ENTERED;
    if (!allowContractCalls && !whitelistedContracts[msg.sender]) {
        if (msg.sender != tx.origin) revert NOT_EOA(msg.sender);
    }
    _;
    _GENERAL_LOCK = _NOT_ENTERED;
}

modifier onlyWhitelistedToken(address token) {
+   require(_GENERAL_LOCK == _NOT_ENTERED, "Reentrancy attack detected");
    _GENERAL_LOCK = _ENTERED;
    if (!whitelistedTokens[token]) revert TOKEN_NOT_WHITELISTED(token);
    _;
    _GENERAL_LOCK = _NOT_ENTERED;
}
```
