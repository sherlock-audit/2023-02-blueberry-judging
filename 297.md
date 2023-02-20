Jeiwan

medium

# Users may borrow more than the `MaxLTV` value of a strategy

## Summary
Users may trick `IchiVaultSpell` to remove a portion of their collateral and bring their position above the `MaxLTV` value of the strategy they opened position in. This allows them borrow more than allowed by the strategy.
## Vulnerability Detail
When opening a position in `IchiVaultSpell`, there's a [strict check](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L149) that disallows the LTV of the user's position to go above the maximal LTV of the strategy. When removing via the [reducePosition](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L266) function, the check is also enforced. However, users are free to specify any strategy ID since it's not validated: a user may open a position in a strategy with a low `MaxLTV` and then remove a portion of collateral by specifying a strategy with a higher `MaxLTV`.

Each strategy is [identified](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L266) by the vault and the maximal position size. Each strategy also has the [list of supported collateral tokens](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L95) and respective maximal LTV for each token. Thus, it's possible that there are different strategies that support the same collateral tokens. One strategy may have a higher maximal LTV for a collateral token, while another strategy may have a lower maximal LTV for the same collateral token.

[Opening a position](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L166) requires specifying a strategy ID: [strategy defines](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L138) what vault the borrowed funds of the position will be deposited into. When [validating LTV](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L149) during a position opening, the [debt value and the collateral value](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L102-L104) of the position being opened is taken into consideration, which assumes the link between the position and the strategy. However, this link is not preserved in the `reducePosition` function, which withdraws collateral and checks the LTV of the position against an arbitrary strategy.
## Impact
Users can borrow more tokens than specified by the maximal LTV of a strategy.
## Code Snippet
[IchiVaultSpell.sol#L266-L274](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L266-L274)
[IchiVaultSpell.sol#L129-L149](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L129-L149)
## Tool used
Manual Review
## Recommendation
Consider validating the `strategyId` value in the `reducePosition` function: the LTV check should be made against the same strategy the positing was initially opened in.