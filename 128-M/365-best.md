joestakey

medium

# `WichiFarm.burn` allows user to steal harvest of other users

## Summary
Users of `WichiFarm` can steal rewards of other users

## Vulnerability Detail
`burn` calls `farm.harvest`, and sends the rewards to the caller. The issue is that the harvesting includes rewards of other users, as the harvesting is done for `address(this)`.

## Impact
Theft of yield

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/wrapper/WIchiFarm.sol#L128

## Tool used
Manual Review

## Recommendation
Account each user deposits when transferring rewards