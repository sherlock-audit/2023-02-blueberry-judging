carrot

high

# IchiLP token pricing mechanism vulnerable to price manipulation

## Summary
The IchiLP tokens are priced as r0*px0 + r1*px1, where r0,r1 are the reserves, and px0,px1 are the oracle prices. This pricing formula has been the cause of multiple price manipulation exploits, like [Warp Finance](https://cmichel.io/pricing-lp-tokens/)
## Vulnerability Detail
The IchiLpOracle contract prices the LP tokens in the manner shown
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/IchiLpOracle.sol#L27-L38

The Ichi LP tokens are priced according to the formula
$$value = r0 * px0 + r1 * px1$$
where $r0,r1$ are the token reserves, and $px0,px1$ are the oracle prices. AMMs also follow the invariant formula $r0 * r1 = K$, which is also true for UniswapV3 when we consider the real reserves. This implies that $r0$ can be written as $K/r1$ and thus the pricing formula becomes
$$value = K * px0 / r1 + r1 * px1$$
which is a formula dependent solely on r1 (assuming oracle prices cannot be manipulated), and thus the composition of the uniV3 pool, which can be changed within a transaction with a flashloan.
Plotting out the value over $r1$, we get a curve like shown.
![image](https://user-images.githubusercontent.com/108595272/219832116-d48790e6-4906-47c5-a2e9-7b904cb7c528.png)
If the composition of the pool is close to this inflection point, the attacker can change the value of the LP tokens at a very low cost, leading to bad debt in the protocol.


## Impact
Oracle contract susceptible to manipulation
## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/IchiLpOracle.sol#L27-L38
## Tool used

Manual Review

## Recommendation
Use the LP token pricing formula established by alphaventure dao, which uses LP invariants. Can be seen discussed [here](https://cmichel.io/pricing-lp-tokens/) and [here](https://blog.alphaventuredao.io/fair-lp-token-pricing/)