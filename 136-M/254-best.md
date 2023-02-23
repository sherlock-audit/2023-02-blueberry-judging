tives

high

# IchiLpOracle returns inflated price due to invalid calculation

## Summary

`IchiLpOracle` returns inflated price due to invalid calculation

## Vulnerability Detail

If you run the tests, then you can see that IchiLpOracle returns inflated price for the ICHI_USDC vault

```solidity
STATICCALL IchiLpOracle.getPrice(token=0xFCFE742e19790Dd67a627875ef8b45F17DB1DaC6) => (1101189125194558706411110851447)
```

As the documentation says, the token price should be in USD with 18 decimals of precision. The price returned here is `1101189125194_558706411110851447` This is 1.1 trillion USD when considering the 18 decimals. 

The test uses real values except for mocking ichi and usdc price, which are returned by the mock with correct decimals (1e18 and 1e6)

## Impact

`IchiLpOracle` price is used in `_validateMaxLTV` (collToken is the vault). Therefore the collateral value is inflated and users can open bigger positions than their collateral would normally allow.

## Code Snippet

```solidity
/**
 * @notice Return lp token price in USD, with 18 decimals of precision.
 * @param token The underlying token address for which to get the price.
 * @return Price in USD
 */
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
[link](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/IchiLpOracle.sol/#L38)

## Tool used

Manual Review

## Recommendation

Fix the LP token price calculation. The problem is that you multiply totalReserve with extra 1e18 (`return (totalReserve * 1e18) / totalSupply;)`.