obront

medium

# If any incorrect parameters are added to a new bank, token will be permanently locked out of the protocol

## Summary

When new tokens are added to the protocol with the `addBank()` function, there aren't adequate checks on the parameters. These tokens are permanently added, many values are unchangeable, and the tokens added cannot be added again. The result is that any incorrect parameters will permanently lock these tokens from the platform.

## Vulnerability Detail

When the admins add a new token to the protocol, they call the `addBank()` function:

```solidity
function addBank(
    address token,
    address cToken,
    address softVault,
    address hardVault
    // @ok if oracle could stop supporting a token, it could still be added? - wont be accepted, admin error
) external onlyOwner onlyWhitelistedToken(token) {
    if (
        token == address(0) ||
        cToken == address(0) ||
        softVault == address(0) ||
        hardVault == address(0)
    ) revert ZERO_ADDRESS();
```
The function takes in the token address, cToken address, soft and hard vaults. The only logic check on these parameters is that they are not `address(0)`.

However, once these assets are set, the `token` and `cToken` are no longer allowed to be attached to a new bank.

```solidity
if (cTokenInBank[cToken]) revert CTOKEN_ALREADY_ADDED();
if (bank.isListed) revert BANK_ALREADY_LISTED();
```
Similarly, the hard and soft vaults cannot be changed once they are set.

The result is that, if a token is added with an incorrect cToken, incorrect soft vault, or incorrect hard vault, these parameters cannot be changed, and the token cannot be edited, deleted or replaced, permanently blocking it from being used on the platform.

The same issue exists if, for any reason, the address of a cToken changed, or a token upgrades to a new address but uses the same cToken address, or any other changes occur that require these values to be shifted.

Note: I am aware that admin error is usually precluded from Sherlock contests, but this is not a "setup" task. It is an ongoing risk that could come from unlikely external events or from a small admin error, where any issue is unfixable and would cause major lasting damage. It is important that this possibility does not exist to ensure the protocol can function properly.

## Impact

Important tokens could be locked from being allowed as an underlying or debt asset on the Blueberry protocol.

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L190-L217

## Tool used

Manual Review

## Recommendation

Add a function for admins to update existing banks, changing the cToken, softVault or hardVault to ensure these markets are able to stay live and active.