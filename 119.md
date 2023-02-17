Ruhum

high

# Excess underlying tokens withdrawn through the BlueBerryBank from the SoftVault will be locked up

## Summary
If the user receives more underlying tokens through `BlueBerryBank.withdrawLend()` than they deposited, the excess tokens will be locked up in the contract.

## Vulnerability Detail
When a user lends tokens through the BlueBerryBank contract, the tokens are deposited into the SoftVault and the user is awarded shares in proportion to their deposit. The SoftVault deposits those tokens into the protocol's Compound fork. The vault will earn a yield on their deposit. When the user tries to withdraw their deposit, their shares will be worth more underlying tokens than they initially deposited. Those excess tokens are withdrawn from the vault but are not sent to the caller. Instead, they are locked up in the BlueBerryBank contract.

## Impact
Loss of user funds.

## Code Snippet
When the user lends tokens they are awarded SoftVault shares: https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L620-L662
```sol
    function lend(address token, uint256 amount)
        external
        override
        inExec
        poke(token)
        onlyWhitelistedToken(token)
    {
        if (!isLendAllowed()) revert LEND_NOT_ALLOWED();

        Position storage pos = positions[POSITION_ID];
        Bank storage bank = banks[token];
        if (pos.underlyingToken != address(0)) {
            // already have isolated collateral, allow same isolated collateral
            if (pos.underlyingToken != token)
                revert INCORRECT_UNDERLYING(token);
        }

        IERC20Upgradeable(token).safeTransferFrom(
            pos.owner,
            address(this),
            amount
        );
        amount = doCutDepositFee(token, amount);
        pos.underlyingToken = token;
        pos.underlyingAmount += amount;

        if (address(ISoftVault(bank.softVault).uToken()) == token) {
            IERC20Upgradeable(token).approve(bank.softVault, amount);
            pos.underlyingVaultShare += ISoftVault(bank.softVault).deposit(
                amount
            );
        } else {
            IERC20Upgradeable(token).approve(bank.hardVault, amount);
            pos.underlyingVaultShare += IHardVault(bank.hardVault).deposit(
                token,
                amount
            );
        }

        bank.totalLend += amount;

        emit Lend(POSITION_ID, msg.sender, token, amount);
    }
```

In `withdrawLend()` the user's tokens are withdrawn from the SoftVault. If the amount is higher than their original deposit (`pos.underlyingAmount`), the value is overwritten with the original deposit: https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L669-L704
```sol
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
The excess funds, `wAmount - pos.underlyingAmount` will be stuck in the BlueBerryBank contract.

## Tool used

Manual Review

## Recommendation
If `wAmount > pos.underlyingAmount`, the whole amount should be sent to the user. Set `pos.underlyingAmount` to `0` to prevent an underflow.
