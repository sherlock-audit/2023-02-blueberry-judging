GimelSec

high

# Should not use Mockup contracts in production

## Summary

Should not use Mockup contracts in production. The mockup contracts will break the protocol.

## Vulnerability Detail

```solidity
oracle/UniswapV3AdapterOracle.sol
13:import "../libraries/UniV3/UniV3WrappedLibMockup.sol";
54:        (int24 arithmeticMeanTick, ) = UniV3WrappedLibMockup.consult(
58:        uint256 quoteTokenAmountForStable = UniV3WrappedLibMockup

spell/IchiVaultSpell.sol
13:import "../libraries/UniV3/UniV3WrappedLibMockup.sol";
313:                    ? UniV3WrappedLibMockup.MAX_SQRT_RATIO - 1 // Token0 -> Token1
314:                    : UniV3WrappedLibMockup.MIN_SQRT_RATIO + 1, // Token1 -> Token0
```

In `oracle/UniswapV3AdapterOracle.sol`, the `UniV3WrappedLibMockup` contract doesn't implement `consult()` and `getQuoteAtTick()`, which will break the oracle.

## Impact

The mockup contracts will break the protocol.

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/UniswapV3AdapterOracle.sol#L54-L64
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L313-L314

## Tool used

Manual Review

## Recommendation

Remove mockup contracts.
