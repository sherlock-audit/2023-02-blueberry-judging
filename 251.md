HonorLt

medium

# Restart fee window

## Summary

The vault withdrawal fee period can be restarted anytime

## Vulnerability Detail

As the name suggests, `startVaultWithdrawFee` should only be called once, however, the code has no protection from calling this function as many times as the owner wants:
```solidity
    function startVaultWithdrawFee() external onlyOwner {
        withdrawVaultFeeWindowStartTime = block.timestamp;
    }
```

This means that once the initial fee period ends, it is possible to start it over again and again.

## Impact

Now it is possible to restart the fee period anytime even a long time after the initial launch. This leaves the opportunity to collect fees from unsuspected users even though the documents and parameters say the fee is enforced only during the initial bootstrap period.
This gives a huge disadvantage and distrust from the perspective of users. 

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/ProtocolConfig.sol#L43-L45

## Tool used

Manual Review

## Recommendation

The code should prevent repetitive initialization of `withdrawVaultFeeWindowStartTime`. Once set it must not be changed. However, if this is an intended behavior, then clearly document and inform the users.
