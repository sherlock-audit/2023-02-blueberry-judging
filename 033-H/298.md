ctf_sec

high

# IchiLpOracle is vulnerable to manipulation

## Summary

IchiLpOracle is vulnerable to manipulation

## Vulnerability Detail

The IChiLpOracle.sol is very vulnerable to manipulation:

```solidity
contract IchiLpOracle is UsingBaseOracle, IBaseOracle {
    constructor(IBaseOracle _base) UsingBaseOracle(_base) {}

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
}
```

One of the way is to directly manipulate the vault.totalSupply()

if the large amount of vault share is minted, totalSupply goes up, and the price goes down based on the code:

```solidity
return (totalReserve * 1e18) / totalSupply;
```

then the getPrice returns a low price, which allows the malicious user to over-borrow and never pay back the debt

The user could borrow flashloan, deposit into the vault, and make the ICHiLPOracle.sol return low price, and make the collaterallized asset under-collateralized and perform malicious liquidation and they withdraw from the vault to burn the share.

Another manipulation path is by exploiting the code:

```solidity
(uint256 r0, uint256 r1) = vault.getTotalAmounts();
```

which calls the code below (I use the Mock vault as a reference)

```solidity
    /**
     @notice Calculates total quantity of token0 and token1 in both positions (and unused in the ICHIVault)
     @param total0 Quantity of token0 in both positions (and unused in the ICHIVault)
     @param total1 Quantity of token1 in both positions (and unused in the ICHIVault)
     */
    function getTotalAmounts()
        public
        view
        override
        returns (uint256 total0, uint256 total1)
    {
        (, uint256 base0, uint256 base1) = getBasePosition();
        (, uint256 limit0, uint256 limit1) = getLimitPosition();
        total0 = IERC20(token0).balanceOf(address(this)) + base0 + limit0;
        total1 = IERC20(token1).balanceOf(address(this)) + base1 + limit1;
```

Since the vault use balanceOf directly, the user can transfer the token0 and token1 directly to the vault and inflate the LP price based on the math:

```solidity
 uint256 totalReserve = (r0 * px0) /
            10**t0Decimal +
            (r1 * px1) /
            10**t1Decimal;

        return (totalReserve * 1e18) / totalSupply;
```

If r0 and r1 goes up, totalReserve goes up, totalSupply does not change, price is inflated.

Then user can over-borrow by using ICHI LP as collateral, the user can still use withdraw on the vault to get the transferred token0 and token1 back.

## Impact

Oracle manipulation in IChiLPOracle allows user to over-borrow or perform malicious liquidation.

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/IchiLpOracle.sol#L18-L39

## Tool used

Manual Review

## Recommendation

We recommend the protocol does not use spot value to derive the oracle price, the spot value used including the vault share total supply and the r0 and r1amount

```solidity
(uint256 r0, uint256 r1) = vault.getTotalAmounts();
```
