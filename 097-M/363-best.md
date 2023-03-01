ctf_sec

medium

# Underlying exchange rate can be manipulated in compound (fork), which impacting the mint / redeem in SoftVault.sol

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

the amount token redeem is determined by:

```solidity
  /* exchangeRate = invoke Exchange Rate Stored() */
  Exp memory exchangeRate = Exp({mantissa: exchangeRateStoredInternal() });
```

which calls:

```solidity
    /**
     * @notice Calculates the exchange rate from the underlying to the CToken
     * @dev This function does not accrue interest before calculating the exchange rate
     * @return calculated exchange rate scaled by 1e18
     */
    function exchangeRateStoredInternal() virtual internal view returns (uint) {
        uint _totalSupply = totalSupply;
        if (_totalSupply == 0) {
            /*
             * If there are no tokens minted:
             *  exchangeRate = initialExchangeRate
             */
            return initialExchangeRateMantissa;
        } else {
            /*
             * Otherwise:
             *  exchangeRate = (totalCash + totalBorrows - totalReserves) / totalSupply
             */
            uint totalCash = getCashPrior();
            uint cashPlusBorrowsMinusReserves = totalCash + totalBorrows - totalReserves;
            uint exchangeRate = cashPlusBorrowsMinusReserves * expScale / _totalSupply;

            return exchangeRate;
        }
    }
```

According to https://docs.compound.finance/v2/ctokens/#exchange-rate

Each cToken is convertible into an ever increasing quantity of the underlying asset, as interest accrues in the market. The exchange rate between a cToken and the underlying asset is equal to:

```solidity
exchangeRate = (getCash() + totalBorrows() - totalReserves()) / totalSupply()
```

given there is no slippage control and deadline check of the cToken.redeem, before the SoftVault.sol.withdraw transaction landed, the transaction happens before can impact the exchangeRate and impact the amount of redeemed.

For example, the user means to call SoftVault.sol#withdraw at exchange rate 100000 unit, but a user intentionally or unintentionally execute redeem / mint function, which change the exchangeRate, the user has to redeem in less than 100000 unit exchange rate.

## Impact

Exchange rate manipulation or fluacutation impacting cToken mint / redeem

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/vault/SoftVault.sol#L88-L123

## Tool used

Manual Review

## Recommendation

We recommend the protocol add deadline check and slippage check to avoid exchange rate manipulation.