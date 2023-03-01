0Kage

medium

# `withdrawVaultFeeStartTime` can be changed to create loss proportionate to withdrawal fee for soft & hard vault users

## Summary
In ProtocolConfig, `startVaultWithdrawFee` function allows owner to set `start` time of withdrawal fee - any withdrawals within the `withdrawalFeeWindow` period from `start` time are subject to a 1% withdrawal fee.

Although Sherlock policy specifies that `admin` is a multisig & that direct protocol rug pulls are not considered as valid issue, I am submitting this issue for 2 reasons:

- Even an accidental call can cause irreversible, large scale loss (equivalent to withdrawal fee) to almost every user of the protocol
- A clear inconsistency from security perspective - `withdrawalFeeWindow` (the other variable that determines withdrawal fee for users in the same contract) is implemented to be more restrictive than `startTime`. While `startTime` can be changed by owners any number of times, withdrawal window can never be changed once initialized even by owners.
 

## Vulnerability Detail
[`startVaultWithdrawalFee`](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/ProtocolConfig.sol#L44) sets the withdrawal fee window start time as `block.timestamp`. Since time is a monotonically increasing variable, there is no way to correct any change made to the start time by even by owners.

Since every user has funds in `Soft` and/or `Hard` vaults, a call to this function can unfairly charge withdrawal fee from every user, even very old users who are long past the withdrawal fee window.

On the other hand, the other variable that also determines withdrawal fee, `withdrawVaultFeeWindow` is initialized to `60 days`. Once initialized, there is no way to increase/decrease fee window even by owners.

If protocol devs thought that owners should not control `withdrawVaultFeeWindow`, why is such a restrictive stance not implemented for `startTime`? This design choice is inconsistent & perplexing with security implications - at the very least, changing vault withdrawal start time should be as restrictive as changing withdrawal fee window (ideally, changing start time should be more restrictive than changing withdrawal window).

## Impact
Consider the following

- Vault deployed at T = 0, with withdrawal period = 60 days
- Alice has posted 1000 USDC collateral at T = 0
- Alice takes 100 USDC collateral at T = 75 days with 0 withdrawal charges
- At T=75 days, owner calls the `startVaultWithdrawFee` function (accidentally or otherwise)
- Alice takes out 100 USDC collateral at T = 80 days, Alice receives 99 USDC, treasury receives 1 USDC

This is similar to a `retrospective` tax on Alice - even though Alice didn't pay withdrawal fee for earlier withdrawal at T=75 days, she has to pay for fresh withdrawal at T=90 days. Note that Alice position hasn't changed at all during this period

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/ProtocolConfig.sol#L43


## Tool used
Manual Review

## Recommendation

Recommend one of the following:

A. Initialize start time & remove `startVaultWithdrawFee`
B. If `startVaultWithdrawFee` is needed to allow for a deferred start, make sure that it is set only once during lifetime of a vault. Once set, this cannot be changed even by owners
