ctf_sec

medium

# SoftVault cToken redeem can fail if the underlying token has liquidity shortfall in Compound (fork)

## Summary

SoftVault cToken redeem can fail if the underlying token shortfalls

## Vulnerability Detail

In the current implementation, the Soft Vault replies on compound based implementation to do lend and redeem

```solidity
    /**
     * @notice Withdraw underlying assets from Compound
     * @param shareAmount Amount of cTokens to redeem
     * @return withdrawAmount Amount of underlying assets withdrawn
     */
    function withdraw(uint256 shareAmount)
        external
        override
        nonReentrant
        returns (uint256 withdrawAmount)
    {
        if (shareAmount == 0) revert ZERO_AMOUNT();

        _burn(msg.sender, shareAmount);

        uint256 uBalanceBefore = uToken.balanceOf(address(this));
        if (cToken.redeem(shareAmount) != 0) revert REDEEM_FAILED(shareAmount);
        uint256 uBalanceAfter = uToken.balanceOf(address(this));
```

note the function call

```solidity
if (cToken.redeem(shareAmount) != 0) revert REDEEM_FAILED(shareAmount);
```

https://github.com/compound-finance/compound-protocol/blob/a3214f67b73310d547e00fc578e8355911c9d376/contracts/CErc20.sol#L60

which calls:

```solidity
function redeem(uint redeemTokens) override external returns (uint) {
    redeemInternal(redeemTokens);
    return NO_ERROR;
}
```

which calls:

https://github.com/compound-finance/compound-protocol/blob/a3214f67b73310d547e00fc578e8355911c9d376/contracts/CToken.sol#L456

```solidity
  function redeemInternal(uint redeemTokens) internal nonReentrant {
      accrueInterest();
      // redeemFresh emits redeem-specific logs on errors, so we don't need to
      redeemFresh(payable(msg.sender), redeemTokens, 0);
  }
```

which calls:

```solidity
    function redeemFresh(address payable redeemer, uint redeemTokensIn, uint redeemAmountIn) internal {
        require(redeemTokensIn == 0 || redeemAmountIn == 0, "one of redeemTokensIn or redeemAmountIn must be zero");

        /* exchangeRate = invoke Exchange Rate Stored() */
        Exp memory exchangeRate = Exp({mantissa: exchangeRateStoredInternal() });

        uint redeemTokens;
        uint redeemAmount;
        /* If redeemTokensIn > 0: */
        if (redeemTokensIn > 0) {
            /*
             * We calculate the exchange rate and the amount of underlying to be redeemed:
             *  redeemTokens = redeemTokensIn
             *  redeemAmount = redeemTokensIn x exchangeRateCurrent
             */
            redeemTokens = redeemTokensIn;
            redeemAmount = mul_ScalarTruncate(exchangeRate, redeemTokensIn);
        } else {
            /*
             * We get the current exchange rate and calculate the amount to be redeemed:
             *  redeemTokens = redeemAmountIn / exchangeRate
             *  redeemAmount = redeemAmountIn
             */
            redeemTokens = div_(redeemAmountIn, exchangeRate);
            redeemAmount = redeemAmountIn;
        }

        /* Fail if redeem not allowed */
        uint allowed = comptroller.redeemAllowed(address(this), redeemer, redeemTokens);
        if (allowed != 0) {
            revert RedeemComptrollerRejection(allowed);
        }

```

note the function calls:

```solidity
  /* Fail if redeem not allowed */
  uint allowed = comptroller.redeemAllowed(address(this), redeemer, redeemTokens);
  if (allowed != 0) {
      revert RedeemComptrollerRejection(allowed);
  }
```

which calls:

```solidity
    /**
     * @notice Checks if the account should be allowed to redeem tokens in the given market
     * @param cToken The market to verify the redeem against
     * @param redeemer The account which would redeem the tokens
     * @param redeemTokens The number of cTokens to exchange for the underlying asset in the market
     * @return 0 if the redeem is allowed, otherwise a semi-opaque error code (See ErrorReporter.sol)
     */
    function redeemAllowed(address cToken, address redeemer, uint redeemTokens) override external returns (uint) {
        uint allowed = redeemAllowedInternal(cToken, redeemer, redeemTokens);
        if (allowed != uint(Error.NO_ERROR)) {
            return allowed;
        }

        // Keep the flywheel moving
        updateCompSupplyIndex(cToken);
        distributeSupplierComp(cToken, redeemer);

        return uint(Error.NO_ERROR);
    }

    function redeemAllowedInternal(address cToken, address redeemer, uint redeemTokens) internal view returns (uint) {
        if (!markets[cToken].isListed) {
            return uint(Error.MARKET_NOT_LISTED);
        }

        /* If the redeemer is not 'in' the market, then we can bypass the liquidity check */
        if (!markets[cToken].accountMembership[redeemer]) {
            return uint(Error.NO_ERROR);
        }

        /* Otherwise, perform a hypothetical liquidity check to guard against shortfall */
        (Error err, , uint shortfall) = getHypotheticalAccountLiquidityInternal(redeemer, CToken(cToken), redeemTokens, 0);
        if (err != Error.NO_ERROR) {
            return uint(err);
        }
        if (shortfall > 0) {
            return uint(Error.INSUFFICIENT_LIQUIDITY);
        }

        return uint(Error.NO_ERROR);
    }
```

note the check:

```solidity
        /* Otherwise, perform a hypothetical liquidity check to guard against shortfall */
        (Error err, , uint shortfall) = getHypotheticalAccountLiquidityInternal(redeemer, CToken(cToken), redeemTokens, 0);
        if (err != Error.NO_ERROR) {
            return uint(err);
        }
        if (shortfall > 0) {
            return uint(Error.INSUFFICIENT_LIQUIDITY);
        }
```

If the liquidity short fall happens when utilization rate is high, the redeem can revert,

which blocks the withdrawLend function in BlueBerryBank.sol

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
```

## Impact

SoftVault.withdraw revert when redeeming from compound fork if the underlying token has liquidity shortfall in Compound (fork)

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/vault/SoftVault.sol#L88-L123

## Tool used

Manual Review

## Recommendation

We recommend the protocol monitor the liquidation in underlying cToken and set lending cap to avoid locked fund due to liquidatity shortfall.