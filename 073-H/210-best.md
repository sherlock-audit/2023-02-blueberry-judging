clems4ever

high

# Risk free profit out of stategies

## Summary

One can open a new position in the `IchiVaultSpell` using a specific strategy. This strategy is bound to an Ichi vault that can deposit any of the two tokens of the LP. However the borrow token can be any registered and whitelisted token. Therefore a malicious user can use a strategy with a vault supporting Token A and Token B but input Token C as the borrow token and make some risk free profit. 

## Vulnerability Detail

A user can open a position by executing an `openPosition` operation on the IchiVaultSpell. The code is as follows and we can see that the borrow token is simply forwarded to `depositInternal`.

```solidity
function openPosition(
        uint256 strategyId,
        address collToken,
        address borrowToken,
        uint256 collAmount,
        uint256 borrowAmount
    )
        external
        existingStrategy(strategyId)
        existingCollateral(strategyId, collToken)
    {
        // 1-3 Deposit on ichi vault
        depositInternal(
            strategyId,
            collToken,
            borrowToken, <============================================
            collAmount,
            borrowAmount
        );

        // 4. Put collateral - ICHI Vault Lp Token
        address vault = strategies[strategyId].vault;
        doPutCollateral(vault, IERC20(vault).balanceOf(address(this)));
    }
```

Let say the strategy with 0 is for ICHI (token0) / USDC (token1) and let say that MATIC is whitelisted as a borrow token. I can maliciously open a position using the strategy 0 and where I lend some ICHI and borrow some MATIC instead of USDC. The borrow token is not checked in `depositInternal` and the borrow token might not even belong to the LP which has as a side effect to record a debt in MATIC but requiring a deposit in LP in USDC. 

```solidity
function depositInternal(
        uint256 strategyId,
        address collToken,
        address borrowToken,
        uint256 collAmount,
        uint256 borrowAmount
    ) internal {
        Strategy memory strategy = strategies[strategyId];

        // 1. Lend isolated collaterals on compound
        doLend(collToken, collAmount);

        // 2. Borrow specific amounts
        doBorrow(borrowToken, borrowAmount);

        // 3. Add liquidity - Deposit on ICHI Vault
        IICHIVault vault = IICHIVault(strategy.vault);
        bool isTokenA = vault.token0() == borrowToken;
        uint256 balance = IERC20(borrowToken).balanceOf(address(this));
        ensureApprove(borrowToken, address(vault));
        if (isTokenA) {
            vault.deposit(balance, 0, address(this));
        } else {
            vault.deposit(0, balance, address(this));
        }

        // 4. Validate MAX LTV
        _validateMaxLTV(strategyId);

        // 5. Validate Max Pos Size
        uint256 lpPrice = bank.oracle().getPrice(strategy.vault);
        uint256 curPosSize = (lpPrice * vault.balanceOf(address(this))) /
            10**IICHIVault(strategy.vault).decimals();
        if (curPosSize > strategy.maxPositionSize)
            revert EXCEED_MAX_POS_SIZE(strategyId);
    }
```

This would normally revert but, as a malicious user, I can transfer the USDC to the spell.
Let's take actual figures and assume that I want to borrow 1000 MATIC and that 1000 MATIC is worth 1520 USDC and let say that I'm ok to deposit any amount of collateral to cover for the position.

Let's zoom in on the problematic snippet

```solidity
        bool isTokenA = vault.token0() == borrowToken; <====================================================
        uint256 balance = IERC20(borrowToken).balanceOf(address(this));
        ensureApprove(borrowToken, address(vault));
        if (isTokenA) {
            vault.deposit(balance, 0, address(this));
        } else { <======================================================================
            vault.deposit(0, balance, address(this));
        }
```
This code checks whether the borrow token is token A but it assumes that it's token B, which is not the case in this attack scenario.

So the openPosition operation will be executed up until the vault needs to deposit the 1000 MATIC, which it cannot since token1 of the LP is USDC. However, as a malicious user I can transfer the 1000 MATIC to the spell before executing the operation so that this call can be done without reverting. The rest of the function would execute without any issue as long as the collateral covers for the debt and the vault has been approved earlier (which is easy to assume as it can have been done by any other user who have opened a position with USDC as borrow token which has consequently called `ensureApprove`, this is the expected token for this strategy so it's very likely that this would happen). 

At this stage, some amount of ICHI has been deposited as collateral, and 1000 USDC has been invested into the LP and 1000 MATIC are still in the spell. I can very easily execute a `reducePosition` operation in the same transaction to collect this amount which is worth 1520 USDC. I can do that by calling `reducePosition(0, MATIC.address, 0) to collect the 1000 MATIC.

I can now close the position as a normal user and get back my initial collateral investment. This however requires that there is enough funds in the LP to cover for the stolen funds otherwise the following line would revert: 

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L294

All in all, collateral aside, I invested 1000 USDC, and received 1000 MATIC worth 1520 USDC which makes a profit of 520 USDC.

## Impact

Risk free profit out of BlueBerry users. From what I saw in the tests and deployment scripts there is currently only ICHI and USDC supported but this might become a big issue right when a new borrow token is supported.

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L144

## Tool used

Manual Review

## Recommendation

Check that the borrow token is exactly token1 if it's not token0.

```solidity
        uint256 balance = IERC20(borrowToken).balanceOf(address(this));
        ensureApprove(borrowToken, address(vault));
        if (vault.token0() == borrowToken) {
            vault.deposit(balance, 0, address(this));
        } else if (vault.token1() == borrowToken) {
            vault.deposit(0, balance, address(this));
        } else {
            revert UNEXPECTED_BORROW_TOKEN();
        }
```