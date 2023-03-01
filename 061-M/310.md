GimelSec

high

# `BlueBerryBank.addBank` doesn't check whether `token` is the underlying token of `cToken`

## Summary

`BlueBerryBank.addBank()` should check whether `token` is the underlying token of `cToken`.  If the wrong bank is added, users are not able to borrow any `token`.

## Vulnerability Detail

`BlueBerryBank.addBank()` doesn’t check whether `token` is the underlying token of `cToken`.  
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L190
```solidity
    function addBank(
        address token,
        address cToken,
        address softVault,
        address hardVault
    ) external onlyOwner onlyWhitelistedToken(token) {
        …
        Bank storage bank = banks[token];
        if (cTokenInBank[cToken]) revert CTOKEN_ALREADY_ADDED();
        if (bank.isListed) revert BANK_ALREADY_LISTED();
        if (allBanks.length >= 256) revert BANK_LIMIT();
        cTokenInBank[cToken] = true;
        bank.isListed = true;
        bank.index = uint8(allBanks.length);
        bank.cToken = cToken;
        …
    }
```

If `token` is not the underlying token of `cToken`. `BlueBerryBank.doBorrow()` always returns zero.
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L855
```solidity
    function doBorrow(address token, uint256 amountCall)
        internal
        returns (uint256 borrowAmount)
    {
        Bank storage bank = banks[token]; // assume the input is already sanity checked.

        IERC20Upgradeable uToken = IERC20Upgradeable(token);
        uint256 uBalanceBefore = uToken.balanceOf(address(this));
        if (ICErc20(bank.cToken).borrow(amountCall) != 0)
            revert BORROW_FAILED(amountCall);
        uint256 uBalanceAfter = uToken.balanceOf(address(this));

        borrowAmount = uBalanceAfter - uBalanceBefore;
        bank.totalDebt += amountCall;
    }
```

If the wrong bank is added. There is no way to fix the error. Since every `token` and `cToken` can only be added once due to the following code:
```solidity
        if (cTokenInBank[cToken]) revert CTOKEN_ALREADY_ADDED();
        if (bank.isListed) revert BANK_ALREADY_LISTED();
        if (allBanks.length >= 256) revert BANK_LIMIT();
        cTokenInBank[cToken] = true;
        bank.isListed = true;
```

Therefore, if the following call is executed, both `USDC` and `CDAI` are not able to be added again.
```solidity
addBank(USDC, CDAI, usdcSoftVault.address, hardVault.address)
```

## Impact

If the wrong bank is added, no one can borrow the added `token`. Moreover, the wrong bank cannot be fixed.

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L190
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L855


## Tool used

Manual Review

## Recommendation

Add the check in the `BlueBerryBank.addBank`

```solidity
require(token == cToken.underlying())
```
