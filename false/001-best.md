seeu

high

# ERC20 transfer result in the contract MockWETH.sol is not checked

## Summary

ERC20 transfer result in the contract [MockWETH.sol](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/mock/MockWETH.sol) is not checked

## Vulnerability Detail

[MockWETH.sol](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/mock/MockWETH.sol) fails to check the result of `transfer` function. This function needs to be always checked since it may lead to unexpected results.

## Impact

If `transfer` fails, certain tokens return false rather than reverting. Even when a token returns false and doesn't really complete the transfer, it is still considered a successful transfer.

## Code Snippet

[contracts/mock/MockWETH.sol#L30](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/mock/MockWETH.sol#L30)
```Solidity
payable(msg.sender).transfer(wad);
```

## Tool used

Manual Review

## Recommendation

Check the value returned by `transfer`. Alternatively, you can use safeTransfer from OpenZeppelin's SafeERC20.