0xbrett8571

medium

# Timing attacks- Incorrect Time Check on Fee Window Start.

## Summary
`startVaultWithdrawFee()` function sets the start time of the fee window but does not have any checks to ensure that the current time is not before the start time, this could enable an attacker to set the start time to a date in the past, which could affect the fee calculations.

## Vulnerability Detail
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/ProtocolConfig.sol#L43-L45

In more detail, the `startVaultWithdrawFee()` function is a public function that can be called by anyone, but it has the `onlyOwner` modifier, which means that only the owner of the contract can call it. But when the owner calls this function, it sets the `withdrawVaultFeeWindowStartTime` variable to the current block timestamp.

The `withdrawVaultFee` variable is where the percentage fee charged on withdrawals from the vault, and the `withdrawVaultFeeWindow` variable is the duration of the fee window, the duration of the fee window is added to the start time to calculate the end time of the fee window.

However, the issue here is that the `startVaultWithdrawFee()` function does not have any checks to ensure that the current time is not before the start time, and this means that an attacker can set the `withdrawVaultFeeWindowStartTime` variable to a date in the past, which could affect the fee calculations. 

For example, an attacker could set the start time to a date that is before the launch of the protocol, which would mean that no fees would be charged on withdrawals.

## Impact
If an attacker could gain and set the start time of the fee window to a date in the past, the `startVaultWithdrawFee()` function could enable an attacker to exploit the protocol by manipulating the fee window and fees to their advantage. 

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/ProtocolConfig.sol#L43-L45

## Tool used

Manual Review

## Recommendation
In the `startVaultWithdrawFee()` function, you can add a `require` statement to ensure that the current time is after the start time of the fee window.

The `require` statement checks if the current block timestamp is greater than or equal to the start time of the fee window plus the duration of the fee window. If the `require` statement fails, the function will revert with the error message "Fee window has not started yet", so this ensures that the `startVaultWithdrawFee()` function can only be called after the fee window has started and prevents an attacker or any bad actor from setting the start time to a date in the past.

Implementation sample:
```solidity
function startVaultWithdrawFee() external onlyOwner {
+   require(block.timestamp >= withdrawVaultFeeWindowStartTime + withdrawVaultFeeWindow, "Fee window has not started yet");
    withdrawVaultFeeWindowStartTime = block.timestamp;
}
```