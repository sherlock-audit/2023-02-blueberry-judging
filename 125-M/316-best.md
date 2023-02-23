banditx0x

high

# Bad Debt Attack Using Uniswap v3 Flash Manipulation and Ichi Vault

## Summary
Blueberry allows leveraged liquidity provision on Uniswap via its integration with Ichi. The ability to provide an amount of liquidity greater than your collateral in Uniswap v3 allows a bad debt attack via price manipulation of the Uniswap v3 pool.

Note that this is different (far easier) than a Oracle Manipulation or Uniswap v3 TWAP. Unlike those manipulations, this one only requires a manipulation of a Uniswap v3 pool for 1 block, which can be done easily via a large market order on uniswap.

## Vulnerability Detail

Basic framework of attack: Manipulate Uniswap v3 pool with a large market buy -> Provide undercollateralized liquidity via Blueberry -> Sell into the Blueberry liquidity and profit off the difference -> Liquidity position suffers large losses due to price manipulation, where loss > collateral, resulting in bad debt

There was a similar attack possible on Perpetual Protocol. Both Blueberry Ichi Vault and Perpetual Protocol allow leverage on top of a Uniswap v3 Pool, which creates this unique attack vector. This exploit was confirmed to work and was paid out via ImmuneFi:

https://securitybandit.com/2023/02/07/bad-debt-attack-for-perpetual-protocol/


## Proof of Concept


We will use an example using USDC and Token B. Token B is currently selling at $1 USDC.


1. Attacker does a large `swap` on the USDC-Token_B Uniswap v3 pool, so that token_B goes up 2000% to $20

2. Attacker provides collateral on Blueberry to obtain an undercollateralized liquidity position on Uniswap with` openPosition`. The oracle used in Blueberry would calculate the value of liquidity based on the price of Token_B on Uniswap which has been manipulated this block. 

We can see that the value of the LP token is defined in` IchiLpOracle.sol` `getPrice` function


3. Attacker sell ICHI on Uniswap v3. Due to the extra liquidity provided via step 2, they are getting a price better than they originally bought it for.

4.The Liquidity provided from range [1,10] has been converted completely to ICHI. Now the average buy order was at $10, 10x the fair price of ICHI. The PnL of the liquidity position > Collateral, meaning bad debt for Blueberry. 

**Example Math:**

Attacker puts up $13400 of collateral in Blueberry. With a 300% leveraged position, the Blueberry Bank enters a liquidity position worth ~$40000
Token B is trading at 20 USDC. Lets say you deposit 1000 Token_B and 20000 USDC*
The attackers sell order goes into the Blueberry liquidity:
The 20000 USDC in the LP position is converted to 2000 Token_B
Now the LP position has 3000 Token B, butToken B is only worth $1 each. The liquidity is worth $3K

*The liquidity composition would actually depend on Ichi’s liquidity distribution which is constantly changing. 

**Blueberry PnL:** 
Blueberry Bank spent $40k to deposit the liquidity onto Ichi->Uniswap.  
Their liquidity is now worth $3k for a -$37K PnL
The attacker deposited $13K of collateral. Blueberry Bank gets to keep the collateral because the user lost an amount greater than their collateral.
Blueberry PnL = -40000+3000+13000
Overall, they have $26K of bad debt. This is a -200% PnL compared to collateral, breaking the invariant that PnL > -100%

We get a better sell price than buy price due to the extra liquidity provided through the Blueberry/Ichi vault. Note that every gain for the buy selling transaction was lost by the liquidity provider. 

Without leverage, there is no profit for the attacker , because every $ they gain from price discrepancy is lost when they provide liquidity. However, with Blueberry leveraged vaults, they can provide far more liquidity than the collateral they put up, making it possible to perform this attack profitably by causing bad debt to Blueberry. This isn’t a bug on Uniswap or Ichi, it is a business logic bug: Uniswap v3 wasn’t built with security features that safeguard leveraged positions: therefore extra security measures need to be added whenever leverage is added to the system.

## Impact

The bad debt attack could potentially drain all the funds in the borrowable funds in the BlueBerry Bank by directing them into buying liquidity positions at a greatly inflated price with a PnL loss greater than collateral.

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L166-L189


## Tool used

Manual Review

## Recommendation

I mentioned Perpetual Protocol earlier, I recommend the same fix to the problem that they implemented. They introduced a `MAX_PRICE_SPREAD`: a measure of the deviation between the Uniswap v3 pool price and an external oracle price. When `MAX_PRICE_SPREAD` exceeds 20%, deposits into Blueberry/Ichi vaults are not allowed (revert).
