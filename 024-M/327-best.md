berndartmueller

high

# The maximum size of an `ICHI` vault spell position can be arbitrarily surpassed

## Summary

The maximum size of an `ICHI` vault spell position can be arbitrarily surpassed by subsequent deposits to a position due to a flaw in the `curPosSize` calculation.

## Vulnerability Detail

Ichi vault spell positions are subject to a maximum size limit to prevent large positions, ensuring a wide margin for liquidators and bad debt prevention for the protocol.

The maximum position size is enforced in the `IchiVaultSpell.depositInternal` function and compared to the current position size `curPosSize`.

However, the `curPosSize` does not reflect the actual position size, but the amount of Ichi vault LP tokens that are currently held in the `IchiVaultSpell` contract (see [L153](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L153)).

Assets can be repeatedly deposited into an Ichi vault spell position using the `IchiVaultSpell.openPosition` function (via the `BlueBerryBank.execute` function).

On the very first deposit, the `curPosSize` correctly reflects the position size. However, on subsequent deposits, the previously received Ichi vault LP tokens are kept in the `BlueBerryBank` contract. Thus, checking the balance of `vault` tokens in the `IchiVaultSpell` contract only accounts for the current deposit.

### Test case

To demonstrate this issue, please use the following test case:

```diff
diff --git a/test/spell/ichivault.spell.test.ts b/test/spell/ichivault.spell.test.ts
index 258d653..551a6eb 100644
--- a/test/spell/ichivault.spell.test.ts
+++ b/test/spell/ichivault.spell.test.ts
@@ -163,6 +163,26 @@ describe('ICHI Angel Vaults Spell', () => {
                                afterTreasuryBalance.sub(beforeTreasuryBalance)
                        ).to.be.equal(depositAmount.mul(50).div(10000))
                })
+               it("should revert when exceeds max pos size due to increasing position", async () => {
+                       await ichi.approve(bank.address, ethers.constants.MaxUint256);
+                       await bank.execute(
+                               0,
+                               spell.address,
+                               iface.encodeFunctionData("openPosition", [
+                                       0, ICHI, USDC, depositAmount.mul(4), borrowAmount.mul(6) // Borrow 1.800e6 USDC
+                               ])
+                       );
+
+                       await expect(
+                               bank.execute(
+                                       0,
+                                       spell.address,
+                                       iface.encodeFunctionData("openPosition", [
+                                               0, ICHI, USDC, depositAmount.mul(1), borrowAmount.mul(2) // Borrow 300e6 USDC
+                                       ])
+                               )
+                       ).to.be.revertedWith("EXCEED_MAX_POS_SIZE"); // 1_800e6 + 300e6 = 2_100e6 > 2_000e6 strategy max position size limit
+               })
                it("should be able to return position risk ratio", async () => {
                        let risk = await bank.getPositionRisk(1);
                        console.log('Prev Position Risk', utils.formatUnits(risk, 2), '%');
```

Run the test with the following command:

```bash
yarn hardhat test --grep "should revert when exceeds max pos size due to increasing position"
```

The test case fails and therefore shows that the maximum position size can be exceeded **without reverting**.

## Impact

The maximum position size limit can be exceeded, leading to potential issues with liquidations and bad debt accumulation.

## Code Snippet

[spell/IchiVaultSpell.sol#L152-L156](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L152-L156)

```solidity
122: function depositInternal(
123:     uint256 strategyId,
124:     address collToken,
125:     address borrowToken,
126:     uint256 collAmount,
127:     uint256 borrowAmount
128: ) internal {
...      // [...]
147:
148:     // 4. Validate MAX LTV
149:     _validateMaxLTV(strategyId);
150:
151:     // 5. Validate Max Pos Size
152:     uint256 lpPrice = bank.oracle().getPrice(strategy.vault);
153:     uint256 curPosSize = (lpPrice * vault.balanceOf(address(this))) /
154:         10**IICHIVault(strategy.vault).decimals();
155:     if (curPosSize > strategy.maxPositionSize)
156:         revert EXCEED_MAX_POS_SIZE(strategyId);
157: }
```

## Tool used

Manual Review

## Recommendation

Consider determining the current position size using the `bank.getPositionValue()` function instead of using the current Ichi vault LP token balance.
