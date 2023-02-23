tsvetanovv

medium

# Some ERC20 tokens deduct a fee on transfer

## Summary
Some ERC20 token implementations have a fee that is charged on each token transfer. This means that the transferred amount isn't exactly what the receiver will get.

The protocol currently uses these tokens:

> ERC20: [whitelisted - current list of supported assets: USDC, DAI, ALCX, BAL, CRV, ICHI, SUSHI, WBTC, WETH]

If one of this tokens will start charge fee on transfers, the logic will be broken.


## Vulnerability Detail
Many of these tokens use proxy pattern (and USDT too). It's quite probably that in one day one of the tokens will start charge fees. Or you would like to add one more token to whitelist and the token will be with fees.

## Impact
Malicious users can drain the tokens in the contract by constantly creating and canceling trades.

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L22


## Tool used

Manual Review

## Recommendation
Improve support for fee on transfer type of ERC20. When pulling funds from the user using `safeTransferFrom` and `safeTransfer` the usual approach is to compare balances pre/post transfer, like so:
```solidity
uint256 balanceBefore = IERC20(token).balanceOf(address(this));
IERC20(token).transferFrom(msg.sender, address(this), amount);
uint256 transferred = IERC20(token).balanceOf(address(this)) - balanceBefore;
```