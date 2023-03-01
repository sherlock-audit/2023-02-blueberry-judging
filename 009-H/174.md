carrot

high

# Miscalculation of farmed ICHI rewards

## Summary
The `IchiVaultSpell` contract receives ICHI rewards whenever it calls `burn` function of `WIchiFarm` contract. It correctly distributes rewards to the user when they call `closePositionFarm`. However it forgets to distribute rewards when user calls `openPositiongFarm` to increase their position. Those rewards are instead collected by whoever calls `closePositionFarm` next time.
## Vulnerability Detail
The function`openPositionFarm` has two purposes:
1. Open a new farming position
2. Increase an existing farming position - Due to this, the contract implements logic to withdraw the old ERC1155 tokens, and issue fresh new ones reflecting the increased position.

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L235-L242

If a user tries to increase their position using this method, we see that the spell burns the old ERC1155 tokens by calling the `burn` function on the `WIchiFarm` contract. When burning a position, the `WIchiFarm` contract returns the accumulated ICHI rewards as can be seen below

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/wrapper/WIchiFarm.sol#L146-L148

These rewards are meant to be distributed to the holder of this position. However the contract forgets to distribute these rewards and instead they are kept in the Spell contract.

The rewarding mechanism is correctly implemented in the `closePosition` contract, as can be seen here

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L391-L404

Thus a user increasing their position forfeits all rewards collected up to that point, which are instead credited to whoever calls `closePositionFarm` next.
## Impact
Misaccounting of ICHI rewards of users
## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L240-L241
Snippet shows burn without reward distribution
## Tool used

Manual Review

## Recommendation
Implement the same reward distribution mechanism in `openPositionFarm` as it is in `closePositionFarm`