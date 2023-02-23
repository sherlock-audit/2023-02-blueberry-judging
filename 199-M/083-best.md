0xbrett8571

medium

# Timestamp Dependence Manipulation in `Withdraw` Vault Fee Window.

## Summary
Using block "timestamps" for checking the start time and duration of the withdraw vault fee window, block timestamps can be manipulated by a miner. So, it clearly means that the start time and duration of the `withdraw` vault fee window can be altered by a malicious miner, potentially allowing them to receive a larger fee than intended.

## Vulnerability Detail
let us look into this.
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/vault/HardVault.sol#L103-L117
It uses the `block.timestamp` to check if the current time is within the withdraw vault fee window, but block timestamps can be manipulated by a miner, making this method insecure.

Use of `blocktimestamps` to determine the start time and duration of the `withdraw` vault fee window but note that block `timestamps` can be easily manipulated by a miner. As a result, a miner could extend or shorten the fee window as he pleases in order to affect the fee charged to users when they withdraw their assets.

## Impact
This is problematic because block "timestamps" can be manipulated by a miner, making it possible for an attacker to abuse the code logic to bypass the intended rules and restrictions.

For example, a miner could set the block timestamp to a value that is very much earlier than the actual time, making it appear as if the `withdraw` fee window has already started. As a result, the fee would not be applied to the `withdraw` transaction, allowing the miner to benefit from the withdrawal by avoiding the fee.

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/vault/HardVault.sol#L103-L117

## Tool used

Manual Review

## Recommendation
I recommend the following fix.

1. Replace the use of block timestamps for checking the start time and duration of the withdraw vault fee window with a more secure method, such as the block Number.

Use `block number` instead of block timestamps as block numbers are more reliable and less susceptible to manipulation. Here's the updated code for the function `withdraw`:
```solidity
 /**
     * @notice Withdraw underlying assets from Compound
     * @param shareAmount Amount of cTokens to redeem
     * @return withdrawAmount Amount of underlying assets withdrawn
     */
function withdraw(address token, uint256 shareAmount)
    external
    override
    nonReentrant
    returns (uint256 withdrawAmount)
{
    if (shareAmount == 0) revert ZERO_AMOUNT();
    IERC20Upgradeable uToken = IERC20Upgradeable(token);
    _burn(msg.sender, uint256(uint160(token)), shareAmount);
    withdrawAmount = shareAmount;

    // Cut withdraw fee if it is in withdrawVaultFee Window (2 months)
 +  if (block.blockNumber < config.withdrawVaultFeeWindowStartBlock() + config.withdrawVaultFeeWindow()) {
 +     uint256 fee = (withdrawAmount * config.withdrawVaultFee()) / DENOMINATOR;
 +      uToken.safeTransfer(config.treasury(), fee);
 +     withdrawAmount -= fee;
    }
    uToken.safeTransfer(msg.sender, withdrawAmount);

    emit Withdrawn(msg.sender, withdrawAmount, shareAmount);
}
```

2. Consider adding a getter method for `withdrawVaultFeeWindowStartBlock` in the `IProtocolConfig` interface to get the "block number" for the start of the withdraw vault fee window.
```solidity
+ interface IProtocolConfig {
    ...
+    uint256 public withdrawVaultFeeWindowStartBlock;
    ...
+    function withdrawVaultFeeWindowStartBlock() public view returns (uint256);
}
```
So, you can see this way, the actual start block number can be fetched from the implementation of the `IProtocolConfig` contract.