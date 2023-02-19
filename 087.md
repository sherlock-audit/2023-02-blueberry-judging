0xbrett8571

medium

# Missing Reentrancy Lock.

## Summary
There is no protection against reentrancy attacks in the code that uses the `onlyEOAEx` modifier, even though the rest of the code uses the `lock` modifier to guard against reentrancy attacks, this will be a blow in the parts of the code that use the `onlyEOAEx` modifier.

## Vulnerability Detail
At some point, this code uses a reentrancy lock (via the lock modifier) to guard against reentrancy attacks. but vividly there are cases where the lock is not used `(e.g., in the onlyEOAEx modifier)`, which leaves the contract vulnerable to reentrancy attacks.
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L54-L59

## Impact
In the case where the lock modifier is not used, such as in the `onlyEOAEx` modifier. This can allow an attacker to repeatedly call the function, the attacker that compromises the state of the contract can do anything that results in unintended consequences.

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L54-L59

## Tool used

Manual Review

## Recommendation
Add a reentrancy lock to the `onlyEOAEx` modifier, which currently doesn't have any protection against reentrancy attacks.
Add the `lock()` modifier to the `onlyEOAEx` modifier as well.

Example:
```solidity
modifier onlyEOAEx() {
    if (!allowContractCalls && !whitelistedContracts[msg.sender]) {
        if (msg.sender != tx.origin) revert NOT_EOA(msg.sender);
    }
+   lock();
    _;
}
```