chaduke

medium

# IchiVaultSpell.openPosition() will always revert if ICHI Vault Lp Tokens are fees-on-transfer ERC20 tokens.

## Summary
``IchiVaultSpell.openPosition()`` will always revert if ICHI Vault Lp Tokens are fee-on-transfer ERC20 tokens.

## Vulnerability Detail
We show why the ``IchiVaultSpell.openPosition()`` function will always revert if ICHI Vault Lp Tokens are fee-on-transfer ERC20 tokens:

1) First, the ``IchiVaultSpell.openPosition()`` function will call ``depositInternal()`` to lend, borrow, and then store the borrowed tokens in an ICHI vault in exchange for ICHI vault Lp ERC20 tokens, and then call ``doPutCollateral()`` to convert   the ICHI vault Lp ERC20 tokens to wrapped ERC1155 tokens ant put them as collateral in the bank.

```javascript
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
            borrowToken,
            collAmount,
            borrowAmount
        );

        // 4. Put collateral - ICHI Vault Lp Token
        address vault = strategies[strategyId].vault;
        doPutCollateral(vault, IERC20(vault).balanceOf(address(this)));
    }

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

2)  ``doPutCollateral()``  calls ``werc20`` to convert ICHI vault Lp ERC20 tokens to wrapped ERC1155 tokens  and then calls ``bank.putCollateral()`` to store them in the bank. The problem is that for fees-on-transfer ICHI vault Lp ERC20 tokens, the value returned by ``werc20.mint(token, amount)`` might be smaller than ``amount`` and then  ``bank.putCollateral(address(werc20), token, amount)`` will revert due to not sufficient ``amount`` ERC1155 tokens to transfer. The following function does NOT check the return value of ``werc20.mint(token, amount)``, assuming the returned value is equal to ``amount`` as well, which is not true for fees-on-transfer ICHI tokens. 

```javascipt
function doPutCollateral(address token, uint256 amount) internal {
        if (amount > 0) {
            ensureApprove(token, address(werc20));
            werc20.mint(token, amount);
            bank.putCollateral(
                address(werc20),
                uint256(uint160(token)),
                amount
            );
        }
    }
```
3) ``werc20()`` considers the effect of fees-on-transfer input tokens, the amount of minted ERC1155 tokens is ``balanceAfter - balanceBefore``, not the input ``amount``!
 
[https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/wrapper/WERC20.sol#L47-L69](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/wrapper/WERC20.sol#L47-L69)

4) Unfortunately, ``putCollateral()`` still uses ``amount``, attempting to store ``amount`` of ERC1155 tokens in the bank. For fees-on-transfer ICHI tokens, we have ``balanceAfter - balanceBefore < amount``. As a result, ``putCollateral()`` will revert on L106:
[https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L793-L816](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L793-L816)

5) This is because there is no sufficient ERC1155 tokens for ``doERC1155TransferIn()`` to transfer, we only have ``balanceAfter - balanceBefore`` ERC1155 tokens, not ``amount`` of tokens. 
```javascript
function doERC1155TransferIn(
        address token,
        uint256 id,
        uint256 amountCall
    ) internal returns (uint256) {
        uint256 balanceBefore = IERC1155Upgradeable(token).balanceOf(
            address(this),
            id
        );
        IERC1155Upgradeable(token).safeTransferFrom(
            msg.sender,
            address(this),
            id,
            amountCall,
            ""
        );
        uint256 balanceAfter = IERC1155Upgradeable(token).balanceOf(
            address(this),
            id
        );
        return balanceAfter - balanceBefore;
    }
}
``` 
6) As a result, ``doPutCollateral()`` will always revert, so will ``IchiVaultSpell.openPosition()`` if ICHI Vault Lp Tokens are fees-on-transfer ERC20 tokens.

## Impact
 ``IchiVaultSpell.openPosition()`` will always revert if ICHI Vault Lp Tokens are fees-on-transfer ERC20 tokens.

## Code Snippet
See above

## Tool used
VScode
Manual Review

## Recommendation
Revise the ``doPutCollateral()`` function, use the returned value of ``WERC20.mint()`` to decide the amount to store in the bank, do not assume it is ``amount``:

```diff
function doPutCollateral(address token, uint256 amount) internal {
        if (amount > 0) {
            ensureApprove(token, address(werc20));
-           werc20.mint(token, amount);
+         uint256 erc1155Amt = werc20.mint(token, amount);
            bank.putCollateral(
                address(werc20),
                uint256(uint160(token)),
                amount
+               erc1155Amt
            ); 
        }
    }
```