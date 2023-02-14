Bnke0x0

medium

# Usage of deprecated transfer to send ETH

## Summary

## Vulnerability Detail
The original transfer used to send eth uses a fixed stipend of 2300 gas. This was used to prevent reentrancy. However, this limits your protocol to interact with other contracts that need more than that to process the transaction good article about thathttps://consensys.net/diligence/blog/2019/09/stop-using-soliditys-transfer-now/The original transfer used to send eth uses a fixed stipend of 2300 gas. This was used to prevent reentrancy. However, this limits your protocol to interact with other contracts that need more than that to process the transaction good article about thathttps://consensys.net/diligence/blog/2019/09/stop-using-soliditys-transfer-now/

## Impact
Usage of deprecated transfer Swap can revert.
## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/mock/MockWETH.sol#L30

                 `payable(msg.sender).transfer(wad);`

## Tool used

Manual Review

## Recommendation
You used to call instead. For example

```solidity
    (bool success, ) = msg.sender.call{amount}("");
    require(success, "Transfer failed.");
```