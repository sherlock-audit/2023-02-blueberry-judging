banditx0x

high

# Users That Deposit Into Ichi Spell Can be Unfairly Liquidated by Attacker

## Summary

Ichi Spell does not use a Chainlink or TWAP oracle for liquidity positons, but rather the token price in the Uniswap v3 Pool. An attacker can liquidate other users with flashloans and/or swaps on the Uniswap v3 pool and obtain their tokens at a discount.


## Vulnerability Detail

Liquidations for Ichi Liquidity positions are calculated via the` get Price()` function in IchiOracle. However unlike the other Liquidation Oracles which are designed to be liquidation resistant, eg. depending on Chainlink Oracles, this depends on the price of Uniswap.

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
}
```

It is not a Uniswap TWAP but the Uniswap price at a particular block, which makes price manipulation trivial.

Heres how the attacker can create an unfair liquidation:

1. Innocent Ian deposits $10000 of collateral into Blueberry and enters a 300% leveraged, or $30000 liquidity position on ETH-ICHI pool
2. Attacker Alice does a large market sell of ICHI on Uniswap v3 with 50% price impact.
3. Innocent Ian's ETH-ICHI liquidity is calculated via getPrice() to be far less, and is now liquidatable.
4. Attacker Alice liquidates Innocent Ian's LP position and aquires it at a discount.

## Impact

Liquidity Providers on Blueberry Ichi Spell can be unfairly liquidated by attackers, with the attacker profiting from this transaction.

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/IchiLpOracle.sol#L19-L40

## Tool used

Manual Review

## Recommendation

A TWAP oracle should be used to calculate the value of Uniswap liquidity positions, or liquidations should not happen if the Uniswap price significantly differs from the price of external price oracles.