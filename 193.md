8olidity

medium

# Intialize() existing risk of front-running

## Summary
Intialize() existing risk of front-running
## Vulnerability Detail
IchivaultSpell contracts have no protection. There is a risk of front-running

```solidity
function initialize(
    IBank _bank,
    address _werc20,
    address _weth,
    address _wichiFarm
) external initializer { //@audit  
    __BasicSpell_init(_bank, _werc20, _weth);

    wIchiFarm = IWIchiFarm(_wichiFarm);
    ICHI = address(wIchiFarm.ICHI());
    IWIchiFarm(_wichiFarm).setApprovalForAll(address(_bank), true);
}
```
## Impact
Intialize() existing risk of front-running
## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L59-L70
## Tool used

Manual Review

## Recommendation
Elector who limits the initialize function