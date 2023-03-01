8olidity

medium

# some tokens that don't support approve type(uint256).max amount.

## Summary
some tokens that don't support approve type(uint256).max amount.
## Vulnerability Detail
some tokens that don't support approve `type(uint256).max` amount.`UNI` and `COMP` support type(uint256).max - they both have this code
[https://etherscan.io/token/0x1f9840a85d5af5bf1d1762f925bdaddc4201f984#code](https://etherscan.io/token/0x1f9840a85d5af5bf1d1762f925bdaddc4201f984#code)

```solidity
function approve(address spender, uint rawAmount) external returns (bool) {
        uint96 amount;
        if (rawAmount == uint(-1)) {
            amount = uint96(-1);
        } else {
            amount = safe96(rawAmount, "Uni::approve: amount exceeds 96 bits");
        }

        allowances[msg.sender][spender] = amount;

        emit Approval(msg.sender, spender, amount);
        return true;
    }
```
Then the `ensureapprove()` function does not work, because it cannot be licensed to `type(uint256).max`

Reference: https://github.com/sherLock-Audit/2022-11-dodo-judging/issues/41


## Impact
some tokens that don't support approve type(uint256).max amount.
## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/BasicSpell.sol#L47-L52
## Tool used

Manual Review

## Recommendation
approve only the necessay amount of token to the approveTarget instead of the type(uint256).max amount.