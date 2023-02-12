PRAISE

high

# Anyone can steal isolated collateral tokens in BlueBerryBank.sol.

## Summary
According to this comment  _"* @dev Withdraw isolated collateral tokens lent to bank. Must only be called from spell while under execution_", withdrawLend() in BlueBerryBank.sol **MUST** only be called by **SPELL**.

## Vulnerability Detail
The external function withdrawLend() is used to withdraw isolated collateral tokens lent to bank and is supposed to be called by only SPELL but there's no Access control implemented to make sure that only SPELL can call withdrawLend(), it is callable by anyone.
## Impact
Anyone can call withdrawLend(), pass in Isolated collateral token address and the amount share token to withdraw and withdraw isolated collateral tokens lent to bank.
## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L669-L704
## Tool used

Manual Review

## Recommendation

You can use something like this: require(msg.sender == SPELL, 'only spell');, to restrict the access of this function.
