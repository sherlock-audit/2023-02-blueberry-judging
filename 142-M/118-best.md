0xbrett8571

medium

# Unrestricted burn and harvest users can loss their funds.

## Summary
See vulnerability detail.

## Vulnerability Detail
In the `burn()` function, the function does not check whether there are any pending ICHI rewards before harvesting it, in that way an attacker can exploit it by calling the `burn()` function with a large value, causing the contract to harvest and transfer all the ICHI rewards that have not yet been collected.

Breakdown of the function:
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/wrapper/WIchiFarm.sol#L116-L135

I will explain it in detail for a clear understanding.

The `burn()` function takes two arguments, `id` and `amount`, the `id` argument is an ERC1155 token ID, and the `amount` argument is the number of tokens to burn.

So, the first thing the function does is check if `amount` is equal to the maximum value of an unsigned integer, If it is, the function retrieves the balance of the caller's address for the given id. This is done to enable the caller to burn their entire balance of the given ERC1155 token.

The `decodeId()` function is then called to retrieve the pool ID (`pid`) and the Ichi amount per share `(stIchiPerShare)` associated with the id.

Next, the `_burn()` function is called to burn the specified amount of ERC1155 tokens from the caller's balance for the given id.

After that, the function calls the `pendingIchi()` function to retrieve the total amount of uncollected ICHI rewards for the pool ID (`pid`) from the farm contract, this amount is then stored in the `ichiRewards` variable.

The function then calls the `harvest()` function to harvest all the ICHI rewards for the given pool ID (`pid`) and deposit them in the contract address (`address(this)`). It also calls the `withdraw()` function to withdraw the specified amount of LP tokens from the pool.

Finally, if the `ichiRewards` variable is greater than zero, the function calls the `safeApprove()` function on the ICHIv1 contract to approve the transfer of `ichiRewards` tokens to the ICHI contract, It then calls the `convertToV2()` function on the ICHI contract to convert the ICHI tokens to ICHIv2 tokens.

## Impact
Could be used to drain the ICHI rewards, reducing the rewards that other users could earn.

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/wrapper/WIchiFarm.sol#L128
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/wrapper/WIchiFarm.sol#L116-L135
## Tool used

Manual Review

## Recommendation
The following changes should be made to the `burn` function.

1. Check whether there are any pending ICHI rewards before harvesting them, this can be done by calling the `pendingIchi` function to check the amount of rewards that have not been claimed yet.
2. If there are any pending ICHI rewards, transfer them to the caller before burning the tokens, this can be done by calling the `safeTransfer` function on the ICHI token contract.
3. After burning the tokens, transfer the LP tokens back to the caller, this can be done by calling the `safeTransfer` function on the LP token contract.