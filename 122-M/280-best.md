y1cunhui

medium

# `WIchiFarm.burn` may fail with safeApprove

## Summary
The `safeApprove` in function `WIchiFarm.burn` may fail if the `ICHI.convertV2` does not spend all allowance of token `ICHIv1`.
## Vulnerability Detail
As the code snippet shown below, there is no check for `allowance[address(this)][address(ICHI)] == 0`, which may cause the `safeApprove` fail. Since no guarentee that `ICHI.convertV2` will always spend all allowance.
## Impact
This missing check may cause the `burn` function permanently unavailable for some token.
## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/wrapper/WIchiFarm.sol#L132-L135
## Tool used

Manual Review

## Recommendation
Add the check described above