0xbrett8571

medium

# Timestamp Dependence.

## Summary
Using `block.timestamp` to set `withdrawVaultFeeWindowStartTime`, but since the `block.timestamp` can be manipulated by miners, attackers can perform a front-running attack by submitting a transaction to start the window at a later time, and this would give the attackers an advantage in withdrawing funds from the vault.

## Vulnerability Detail
The way `withdrawVaultFeeWindowStartTime` is set, the code sets the value of `withdrawVaultFeeWindowStartTime` using the `block.timestamp`, which is the timestamp of the block that is currently being processed by the network, however this value can be manipulated by miners, and attackers can exploit this by performing a front-running attack.

In particular, an attacker can submit a transaction to start the window at a later time, giving them an advantage in withdrawing funds from the vault. 
For example, suppose an attacker knows that a large withdrawal is going to occur soon, and they want to withdraw their funds before that happens. In that case, they can submit a transaction to start the withdrawal window at a later time and ensure that their withdrawal transaction is processed before the large withdrawal transaction.

## Impact
An attacker can manipulate the `withdrawVaultFeeWindowStartTime` by submitting a transaction at a later time, which will give them an advantage in withdrawing funds from the vault, the attacker can then delay the start time of the window, allowing them enough time to withdraw the funds at a lower fee or even for free.

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/ProtocolConfig.sol#L44

## Tool used

Manual Review

## Recommendation
To avoid manipulation by miners, the contract should use `block.number` instead of `block.timestamp` to record the starting time for the `withdrawVaultFeeWindow`.