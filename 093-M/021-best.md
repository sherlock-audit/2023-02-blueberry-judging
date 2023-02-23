koxuan

medium

# onlyEOAEx modifier that ensures call is from EOA might not hold true in the future

## Summary
modifier `onlyEOAEx` is used to ensure calls are only made from EOA. However, EIP 3074 suggests that using `onlyEOAEx` modifier to ensure calls are only from EOA might not hold true.

## Vulnerability Detail
For `onlyEOAEx`, `tx.origin` is used to ensure that the caller is from an EOA and not a smart contract.
```solidity
    modifier onlyEOAEx() {
        if (!allowContractCalls && !whitelistedContracts[msg.sender]) {
            if (msg.sender != tx.origin) revert NOT_EOA(msg.sender);
        }
        _;
    }

```

However, according to [EIP 3074](https://eips.ethereum.org/EIPS/eip-3074#abstract),

This EIP introduces two EVM instructions AUTH and AUTHCALL. The first sets a context variable authorized based on an ECDSA signature. The second sends a call as the authorized account. This essentially delegates control of the externally owned account (EOA) to a smart contract.

Therefore, using tx.origin to ensure msg.sender is an EOA will not hold true in the event EIP 3074 goes through.



## Impact

Using modifier `onlyEOAEx` to ensure calls are made only from EOA will not hold true in the event EIP 3074 goes through. 

## Code Snippet
[BlueBerryBank.sol#L54-L59](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L54-L59)
## Tool used

Manual Review

## Recommendation

Recommend using OpenZepellin's `isContract` function (https://docs.openzeppelin.com/contracts/2.x/api/utils#Address-isContract-address-). Note that there are edge cases like contract in constructor that can bypass this and hence caution is required when using this.

```solidity
    modifier onlyEOAEx() {
        if (!allowContractCalls && !whitelistedContracts[msg.sender]) {
            if (isContract(msg.sender)) revert NOT_EOA(msg.sender);
        }
        _;
    }

```
