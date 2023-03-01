berndartmueller

high

# Failure to refund `ICHI` v2 farming reward tokens upon increasing farming position

## Summary

`ICHI` v2 farming reward tokens are not refunded to the user when subsequently calling the `IchiVaultSpell.openPositionFarm` function to increase the farming position. Any other user with a farming position can steal those left behind `ICHI` v2 reward tokens.

## Vulnerability Detail

A user can repeatedly call the `IchiVaultSpell.openPositionFarm` function for a given position to increase the farming position. Any existing `WIchiFarm` token collateral previously deposited as part of the position is taken out from the bank and burned via the `WIchiFarm.burn` function in [line 241](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L241).

Burning the wrapped `WIchiFarm` ERC1155 tokens withdraws the underlying LP tokens from the Ichi farm and harvests `ICHI` v1 and v2 reward tokens (see [WIchiFarm.sol#L128-L129](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/wrapper/WIchiFarm.sol#L128-L129)). Those `ICHI` v2 tokens, along with the LP tokens, are then transferred to the spell contract.

The previously withdrawn LP tokens and the newly added LP tokens are then wrapped into `WIchiFarm` tokens to farm rewards, and deposited into the bank as collateral.

However, the `ICHI` v2 reward tokens harvested earlier are not refunded to the user but remain in the spell contract. Consequently, any user can steal these ICHI v2 reward tokens by closing their own farming position.

## Impact

This vulnerability results in the loss of eligible `ICHI` v2 farming reward tokens for users who increase their farming position, which can then be stolen by others.

## Code Snippet

[spell/IchiVaultSpell.sol#L241](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L241)

```solidity
199: function openPositionFarm(
200:     uint256 strategyId,
201:     address collToken,
202:     address borrowToken,
203:     uint256 collAmount,
204:     uint256 borrowAmount,
205:     uint256 farmingPid
206: )
207:     external
208:     existingStrategy(strategyId)
209:     existingCollateral(strategyId, collToken)
210: {
211:     Strategy memory strategy = strategies[strategyId];
212:     address lpToken = wIchiFarm.ichiFarm().lpToken(farmingPid);
213:     if (strategy.vault != lpToken) revert INCORRECT_LP(lpToken);
214:
215:     // 1-3 Deposit on ichi vault
216:     depositInternal(
217:         strategyId,
218:         collToken,
219:         borrowToken,
220:         collAmount,
221:         borrowAmount
222:     );
223:
224:     // 4. Take out collateral
225:     (
226:         ,
227:         ,
228:         ,
229:         ,
230:         address posCollToken,
231:         uint256 collId,
232:         uint256 collSize,
233:
234:     ) = bank.getCurrentPositionInfo();
235:     if (collSize > 0) {
236:         (uint256 decodedPid, ) = wIchiFarm.decodeId(collId);
237:         if (farmingPid != decodedPid) revert INCORRECT_PID(farmingPid);
238:         if (posCollToken != address(wIchiFarm))
239:             revert INCORRECT_COLTOKEN(posCollToken);
240:         bank.takeCollateral(collSize);
241:         wIchiFarm.burn(collId, collSize); // @audit-info returns ICHI vault LP tokens and ICHI v2 token rewards
242:     }
243:
244:     // 5. Deposit on farming pool, put collateral
245:     ensureApprove(strategy.vault, address(wIchiFarm));
246:     uint256 lpAmount = IERC20(strategy.vault).balanceOf(address(this));
247:     uint256 id = wIchiFarm.mint(farmingPid, lpAmount);
248:     bank.putCollateral(address(wIchiFarm), id, lpAmount);
249: }
```

[IchiVaultSpell.closePositionFarm](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L404)

The current `ICHI` v2 token balance of the `IchiVaultSpell` contract is always refunded to the user when closing a farming position in line 404. Any left behind `ICHI` v2 tokens from other users (which are accumulated in this contract due to the aforementioned issue) can be stolen.

```solidity
367: function closePositionFarm(
368:     uint256 strategyId,
369:     address collToken,
370:     address borrowToken,
371:     uint256 lpTakeAmt,
372:     uint256 amountRepay,
373:     uint256 amountLpWithdraw,
374:     uint256 amountShareWithdraw
375: )
376:     external
377:     existingStrategy(strategyId)
378:     existingCollateral(strategyId, collToken)
379: {
...      // [...]
403:     // 9. Refund ichi token
404:     doRefund(ICHI);
405: }
```

## Tool used

Manual Review

## Recommendation

Consider refunding the ICHI v2 token (`ICHI`) rewards to the user at the end of the `IchiVaultSpell.openPositionFarm` function:

```diff
  199: function openPositionFarm(
  200:     uint256 strategyId,
  201:     address collToken,
  202:     address borrowToken,
  203:     uint256 collAmount,
  204:     uint256 borrowAmount,
  205:     uint256 farmingPid
  206: )
  ...      // [...]
  235:     if (collSize > 0) {
  236:         (uint256 decodedPid, ) = wIchiFarm.decodeId(collId);
  237:         if (farmingPid != decodedPid) revert INCORRECT_PID(farmingPid);
  238:         if (posCollToken != address(wIchiFarm))
  239:             revert INCORRECT_COLTOKEN(posCollToken);
  240:         bank.takeCollateral(collSize);
  241:         wIchiFarm.burn(collId, collSize);
  242:     }
  243:
  244:     // 5. Deposit on farming pool, put collateral
  245:     ensureApprove(strategy.vault, address(wIchiFarm));
  246:     uint256 lpAmount = IERC20(strategy.vault).balanceOf(address(this));
  247:     uint256 id = wIchiFarm.mint(farmingPid, lpAmount);
  248:     bank.putCollateral(address(wIchiFarm), id, lpAmount);
  249:
+ 250:     doRefund(ICHI);
  251: }
```

Additionally, as stated in the docs, deduct performance fees from the `ICHI` v2 reward tokens as necessary.
