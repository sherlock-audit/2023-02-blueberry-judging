koxuan

medium

# A user will be unable to close position if he gets blacklisted by collateral asset token contract

## Summary
Take USDC for example, if a user gets blacklisted by it after opening a position, he is unable to be close his position, and is only able to retrieve his collateral if the position becomes too risky where he has to fight with others to liquidate his position.

## Vulnerability Detail
When user closes his position, the collateral token will be sent to the executor, who is the owner of the position which will be the user. However, in the event that the user get blacklisted by the token contract, the collateral token transfer will always revert, causing user to be unable to close his position.

```solidity
    function doRefund(address token) internal {
        uint256 balance = IERC20Upgradeable(token).balanceOf(address(this));
        if (balance > 0) {
            IERC20Upgradeable(token).safeTransfer(bank.EXECUTOR(), balance);
        }
    }

```
## Impact
User collateral will be stuck in protocol in the event he gets blacklisted until it is liquidatable, where he will have to fight with others who can also liquidate his collateral.
## Code Snippet
[IchiVaultSpell.sol#L357-L364](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L357-L364)
[IchiVaultSpell.sol#L394-L401](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L394-L401)
[IchiVaultSpell.sol#L329](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L329)
[BasicSpell.sol#L56-L61](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/BasicSpell.sol#L56-L61)

## Tool used

Manual Review

## Recommendation

Recommend adding a recipient to allow user to specify where the collateral token should be transferred to in the event he gets blacklisted by the collateral contract.
