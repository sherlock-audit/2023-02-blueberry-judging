stent

medium

# Max value for withdrawLend is not correct

## Summary

If `shareAmount` in `withdrawLend` is set to `type(uint256).max` then it is reassigned to the maximum value that can be withdrawn from Compound. Although the token transfers all work the updating of `pos.underlyingAmount` may have an overflow, resulting in a failed tx.

## Vulnerability Detail

This is the code in question in BlueBerryBank:
```solidity
        function withdrawLend(address token, uint256 shareAmount)
            external
            override
            inExec
            poke(token)
        {
            Position storage pos = positions[POSITION_ID];
            Bank storage bank = banks[token];
            if (token != pos.underlyingToken) revert INVALID_UTOKEN(token);
        if (shareAmount == type(uint256).max) {
            shareAmount = pos.underlyingVaultShare;
        }

        uint256 wAmount;
        if (address(ISoftVault(bank.softVault).uToken()) == token) {
            ISoftVault(bank.softVault).approve(
                bank.softVault,
                type(uint256).max
            );
            wAmount = ISoftVault(bank.softVault).withdraw(shareAmount);
        } else {
            wAmount = IHardVault(bank.hardVault).withdraw(token, shareAmount);
        }

        wAmount = wAmount > pos.underlyingAmount
            ? pos.underlyingAmount
            : wAmount;

        pos.underlyingVaultShare -= shareAmount;
        pos.underlyingAmount -= wAmount;
        bank.totalLend -= wAmount;

        wAmount = doCutWithdrawFee(token, wAmount);

        IERC20Upgradeable(token).safeTransfer(msg.sender, wAmount);
    }
```
`shareAmount` is set to `pos.underlyingVaultShare` in the max case. 

1. Suppose a user opens a new position and some time later would like to close it and withdraw all the lend tokens. When opening a position the `lend` function will `pos.underlyingVaultShare` to `lendAmount / exchangeRate` where `lendAmount` is given by the user and `exchangeRate` is the ratio that Compound gives.
2. When the user wants to close their whole position they set `shareAmount = type(uint256).max` which is then set to `pos.underlyingVaultShare = lendAmount / exchangeRate`
3. The correct SoftVault tokens are sent to the SoftVault and the correct amount of cTokens are sent to Compound. Compound then sends back `amount * exchangeRateNew` of lend tokens to the SoftVault which passes them on to the bank.
4. The updates `pos.underlyingVaultShare` using the following forumla: `pos.underlyingVaultShare -= amount * exchangeRateNew` which can be expanded to `pos.underlyingVaultShare -= lendAmount / exchangeRate * exchangeRateNew`. If `exchangeRateNew >  exchangeRate` then `lendAmount / exchangeRate * exchangeRateNew > lendAmount / exchangeRate = pos.underlyingVaultShare` which will cause an overflow.

## Impact

Any smart contract that is dependent on the max assumption (setting `shareAmount = type(uint256).max` means all lend tokens will be returned) may end up getting stuck if the exchange rate increases on Compound.

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L679

## Tool used

Manual Review

## Recommendation

