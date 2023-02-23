Udsen

false

# CHECK THE ALLOWANCE AMOUNT BY THE `msg.sender` to `_spender` FOR THE GIVEN ERC20 TOKEN IS ZERO BEFORE CALLING THE `approve` FUNCTION

## Summary
It is recommended to check allowance from the `msg.sender` to the `spender` for the given erc20 token is zero before calling the `token.approve()` function.

## Vulnerability Detail
Some ERC20 tokens such as `usdt` revert if the allowance > 0 when the approve function is called on the contract.

## Impact
The transaction will revert if the allowance > 0, when approve function is called for ERC20 tokens such as `usdt`

## Code Snippet

```solidity
IERC20Upgradeable(token).approve(bank.cToken, amountCall);
```
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L882

## Tool used

VS Code and Manual Review

## Recommendation

It is recommended to check whether allowance > 0 for `msg.sender` to `_spender` before calling the approve function on the ERC20 token.