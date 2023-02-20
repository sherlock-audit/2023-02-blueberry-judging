ctf_sec

high

# Division before manipulation incurs heavy precision loss in IchiLpOracle

## Summary

Division before manipulation incurs heavy precision loss in IchiLpOracle

## Vulnerability Detail

In the current implementation of the IchiLPOracle, the oracle price is derviced below:

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

the formula is: ((r0 * price0) / 10 ** r0Decimal + r1 * price1) / 10 ** r1Decimal) *  1e18)) / totalSupply,

Division before manipulation incurs heavy precision loss in IchiLpOracle, which truncate the price and make the LP price undervalued

The precision loss happens in  (r0 * px0) / 10**t0Decimal and in  (r1 * px1) / 10**t1Decimal

Then the healthy collateral may be incorrectly liquidated.

## Impact

See above

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/IchiLpOracle.sol#L10-L39

## Tool used

Manual Review

## Recommendation

We recommend the protocol use a manipulation resistant and avoid division before manipulation

I recommend the protocol consider the formula below:

https://switchboardxyz.medium.com/fair-lp-token-oracles-94a457c50239#:~:text=The%20price%20of%20these%20LP,number%20of%20LP%20tokens%20issued.

![image](https://user-images.githubusercontent.com/114844362/220126617-e6b43dfb-3952-4096-9cc8-c53379e2a14e.png)

