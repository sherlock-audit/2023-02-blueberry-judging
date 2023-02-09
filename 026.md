cergyk

high

# User never receive the interest on lending to the protocol

## Summary
Users may lend on the protocol by depositing into soft/hardVaults, which in turn deposit to compound style tokens.
However the user never receives the rewards for lending on compound.

## Vulnerability Detail
When users withdraw from soft-vaults, only the amount the user deposited is refunded:
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L693-L699

wAmount is transferred to the user, which cannot be greater than pos.underlyingAmount, which is the amount of underlying token the user lent to the protocol:
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L644

## Impact
Users do not have access to rewards accumulated by lending.

## Code Snippet

## Tool used

Manual Review

## Recommendation
Do not cap based on underlying amount