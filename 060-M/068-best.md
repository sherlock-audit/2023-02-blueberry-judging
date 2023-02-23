Bnke0x0

medium

# Token ERC20 missing return value check

## Summary

## Vulnerability Detail
ERC20.approve() call but does not check the success return value. Some tokens do not revert if the approval failed but return false instead.

## Impact
Tokens that don't actually perform the approve and return false are still counted as correct approve.
## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L647

                 'IERC20Upgradeable(token).approve(bank.softVault, amount);'

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L652

                           'IERC20Upgradeable(token).approve(bank.hardVault, amount);'

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L684-L687


                                          ' ISoftVault(bank.softVault).approve(
                bank.softVault,
                type(uint256).max
            );'


https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L882


                                            'IERC20Upgradeable(token).approve(bank.cToken, amountCall);'


## Tool used

Manual Review

## Recommendation
I recommend using OpenZeppelin’s [SafeERC20](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/release-v4.1/contracts/token/ERC20/utils/SafeERC20.sol#L74) versions with the safeApprove function that handles the return value check as well as non-standard-compliant tokens.