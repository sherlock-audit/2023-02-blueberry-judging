OCC

high

# Vulnerability in getPrice function to reentrancy attack

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/AggregatorOracle.sol#L89

## Summary
The` getPrice `function may be vulnerable to _a reentrancy attack_ if any of the `primarySources` that are called by `getPrice` allow for external contract calls.

## Vulnerability Detail

A contract containing a `fallback `function that calls the `getPrice` function and then calls back into itself.If the `primarySources` that is called during the execution of the getPrice function also includes a call to an external contract, the `fallback `function could be triggered, allowing them to execute additional code within the `getPrice` function.

## Impact
High

## Code Snippet
Manually Review

## Tool used
Manually Review

### Manual Review
Done

## Recommendation
To prevent reentrancy attacks, we can use _"checks-effects-interactions"_ pattern, which involves performing all checks and state changes before making any external calls. We can also use the `nonReentrant `modifier from the _OpenZeppelin_ library, which can help us to prevents a function from being re-entered by the same caller.
