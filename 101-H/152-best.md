obront

high

# LP tokens cannot be valued because ICHI cannot be priced by oracle, causing all new open positions to revert

## Summary

In order to value ICHI LP tokens, the oracle uses the Fair LP Pricing technique, which uses the prices of both individual tokens, along with the quantities, to calculate the LP token value. However, this process requires the underlying token prices to be accessible by the oracle. Both Chainlink and Band do not support the ICHI token, so the function will fail, causing all new positions using the IchiVaultSpell to revert.

## Vulnerability Detail

When a new Ichi position is opened, the ICHI LP tokens are posted as collateral. Their value is assessed using the `IchiLpOracle#getPrice()` function:

```solidity
function getPrice(address token) external view override returns (uint256) {
    IICHIVault vault = IICHIVault(token);
    uint256 totalSupply = vault.totalSupply();
    if (totalSupply == 0) return 0;

    address token0 = vault.token0();
    address token1 = vault.token1();

    (uint256 r0, uint256 r1) = vault.getTotalAmounts();
    uint256 px0 = base.getPrice(address(token0));
    uint256 px1 = base.getPrice(address(token1));
    uint256 t0Decimal = IERC20Metadata(token0).decimals();
    uint256 t1Decimal = IERC20Metadata(token1).decimals();

    uint256 totalReserve = (r0 * px0) /
        10**t0Decimal +
        (r1 * px1) /
        10**t1Decimal;

    return (totalReserve * 1e18) / totalSupply;
}
```
This function uses the "Fair LP Pricing" formula, made famous by Alpha Homora. To simplify, this uses an oracle to get the prices of both underlying tokens, and then calculates the LP price based on these values and the reserves.

However, this process requires that we have a functioning oracle for the underlying tokens. However, [Chainlink](https://data.chain.link/) and [Band](https://data.bandprotocol.com/) both do not support the ICHI token (see the links for their comprehensive lists of data feeds). As a result, the call to `base.getPrice(token0)` will fail.

All prices are calculated in the `isLiquidatable()` check at the end of the `execute()` function. As a result, any attempt to open a new ICHI position and post the LP tokens as collateral (which happens in both `openPosition()` and `openPositionFarm()`) will revert.

## Impact

All new positions opened using the `IchiVaultSpell`  will revert when they attempt to look up the LP token price, rendering the protocol useless.

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/IchiLpOracle.sol#L19-L39

## Tool used

Manual Review

## Recommendation

There will need to be an alternate form of oracle that can price the ICHI token. The best way to accomplish this is likely to use a TWAP of the price on an AMM.