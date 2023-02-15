mert_eren

high

# A Whale can grieve specific type of borrowing token

## Summary
A Whale can grieve specific type of borrowing token
## Vulnerability Detail
ichiVault token taken with amount of borrowed token in spell contract balance. And total IchiVault token value houldn't be exceed the strategies set value. However, a malicious user can transfer borrow token and cause a revert of other user's openPosition attempt even if their borrowToken amount not exceed the limit. 
## Impact
https://imgur.com/a/XH8NAr6
This test work in ichiVaultSpell.ts
As can seen in photo even if a normal user's openPosiiton attempt revert due to contract try to take much more vault token than the user's borrowAmount.
## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L140-L146
## Tool used

Manual Review

## Recommendation
Try to deposit user's borrowAmount input instead of borrowToken.balanceOf(address(this)) 
