seeu

medium

# Use of transfer() may lead to failures

## Summary

Use of transfer() may lead to failures

## Vulnerability Detail

From [EIP-1884](https://eips.ethereum.org/EIPS/eip-1884), the use of `transfer()` is no longer advisable since it may lead to failures due to the fact that it has hard coded gas budget and can fail when the user is a smart contract.

## Impact

`transfer()` may lead to failures

## Code Snippet

[contracts/mock/MockWETH.sol#L30](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/mock/MockWETH.sol#L30)
```Solidity
payable(msg.sender).transfer(wad);
```

## Tool used

Manual Review

## Recommendation

It is suggested to use `call()` instead of `transfer()` without hardcoded gas limits. Also, implements checks-effects-interactions pattern or reentrancy guards for reentrancy protection.

See the following resources:
- [Stop Using Solidity's transfer() Now](https://consensys.net/diligence/blog/2019/09/stop-using-soliditys-transfer-now/)
- [WithdrawFacet's withdraw calls native payable.transfer, which can be unusable for DiamondStorage owner contract](https://github.com/code-423n4/2022-03-lifinance-findings/issues/14)