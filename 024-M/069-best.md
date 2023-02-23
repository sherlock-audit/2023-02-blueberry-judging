rvierdiiev

medium

# IchiVaultSpell.strategy.maxPositionSize can be bypassed

## Summary
IchiVaultSpell.strategy.maxPositionSize can be bypassed by depositing several times.
## Vulnerability Detail
Protocol is trying to limit risk in the system using the extent of position sizes being created.
For each strategy in the IchiVaultSpell there is maxPositionSize that user should be able to open.

When user borrows and deposits to the ichi vault, then received lp amount [is checked](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L155) to not be bigger than `maxPositionSize`.

But user is not limited to borrow only 1 time for the position. He can borrows as much times as he wishes in case if the borrow is healthy.
So borrower can just take a loan several times, in order to bypass `maxPositionSize` restriction.

This test shows, that user can deposit to ichi vault 2 times in a row with value little bit less, than `maxPositionSize`.
Run it inside `ichivault.spell.test.ts`
```solidity
it.only("should revert when exceeds max pos size", async () => {
			await ichi.approve(bank.address, ethers.constants.MaxUint256);
			await expect(
				bank.execute(
					0,
					spell.address,
					iface.encodeFunctionData("openPosition", [
						//limit in the test is 2000 usd
						//this will open position for 1800
						0, ICHI, USDC, depositAmount.mul(4), borrowAmount.mul(6)
					])
				)
			)

			const nextPosId = await bank.nextPositionId();
			await expect(
				bank.execute(
					nextPosId.sub(1),
					spell.address,
					iface.encodeFunctionData("openPosition", [
						//this will open position for more 1800, so 2000 limit is bypassed
						0, ICHI, USDC, depositAmount.mul(4), borrowAmount.mul(6)
					])
				)
			)

			const pos = await bank.positions(nextPosId.sub(1));
			console.log(pos.collateralSize);
			//position size is 3600
			expect(pos.collateralSize.eq(3600000000)).to.be.true;
		})
```
## Impact
`maxPositionSize` restriction bypass.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
When check `maxPositionSize` limit, mind the position size already put as collateral before current call.