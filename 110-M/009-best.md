0x52

medium

# ChainlinkAdapterOracle use BTC/USD chainlink oracle to price WBTC which is problematic if WBTC depegs

## Summary

The chainlink BTC/USD oracle is used to price WBTC ([docs](https://docs.blueberry.garden/lending-protocol/price-oracle#price-feed-alternative)). WBTC is basically a bridged asset and if the bridge is compromised/fails then WBTC will depeg and will no longer be equivalent to BTC. This will lead to large amounts of borrowing against an asset that is now effectively worthless. Since the protocol still values it via BTC/USD the protocol will not only be stuck with the bad debt caused by the currently outstanding loans but they will also continue to give out bad loans and increase the amount of bad debt further

## Vulnerability Detail

See summary.

## Impact

Protocol will take on a large amount of bad debt should WBTC bridge become compromised and WBTC depegs

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/ChainlinkAdapterOracle.sol#L47-L59

## Tool used

Manual Review

## Recommendation

I would recommend using a double oracle setup. Use both the Chainlink and another on-chain liquidity base oracle (i.e. UniV3 TWAP). If the price of the on-chain liquidity oracle drops below a certain threshold of the Chainlink oracles (i.e. 2% lower), any borrowing should be immediately halted. The chainlink oracle will prevent price manipulation and the liquidity oracle will safeguard against the asset depegging.