berndartmueller

medium

# Rebase/FoT tokens are not supported as isolated collateral

## Summary

The `BlueBerryBank.lend` function does not account for rebase/FoT tokens.

## Vulnerability Detail

The `SoftVault` and `HardVault` contracts are already well prepared to handle rebase/FoT tokens properly. However, the `BlueBerryBank.lend` function does not account for rebase/FoT tokens and will not work properly with them.

As seen in [lines 637-641](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L637-L641), `amount` of the ERC-20 token is transferred from the `pos.owner` to the `BlueBerryBank` contract. From this `amount`, a deposit fee is decremented, and the remaining amount is deposited into the `SoftVault` or `HardVault` contract.

However, if the used `token` is a rebase/FoT ERC-20 token, the received token amount does not reflect the actual amount of tokens transferred. This leads to the incorrect amount of tokens being deposited and accounted for.

While the Blueberry protocol has whitelisting mechanisms in place to restrict the use of arbitrary ERC-20 tokens as isolated collateral, rebase/FoT tokens were likely intended to be supported as isolated collateral due to their support in the `SoftVault` and `HardVault` contracts.

## Impact

Rebase/FoT tokens are not supported as isolated collateral and can not be used as isolated collateral without incurring accounting issues.

## Code Snippet

[BlueBerryBank.sol#L637-L641](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L637-L641)

```solidity
620: function lend(address token, uint256 amount)
621:     external
622:     override
623:     inExec
624:     poke(token)
625:     onlyWhitelistedToken(token)
626: {
627:     if (!isLendAllowed()) revert LEND_NOT_ALLOWED();
628:
629:     Position storage pos = positions[POSITION_ID];
630:     Bank storage bank = banks[token];
631:     if (pos.underlyingToken != address(0)) {
632:         // already have isolated collateral, allow same isolated collateral
633:         if (pos.underlyingToken != token)
634:             revert INCORRECT_UNDERLYING(token);
635:     }
636:
637:     IERC20Upgradeable(token).safeTransferFrom(
638:         pos.owner,
639:         address(this),
640:         amount
641:     );
642:     amount = doCutDepositFee(token, amount);
643:     pos.underlyingToken = token;
644:     pos.underlyingAmount += amount;
645:
...      // [...]
```

[SoftVault.sol#L76](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/vault/SoftVault.sol#L76)

As mentioned above, the `SoftVault` contract calculates the actual amount of tokens received after the transfer in line 75.

```solidity
67: function deposit(uint256 amount)
68:     external
69:     override
70:     nonReentrant
71:     returns (uint256 shareAmount)
72: {
73:     if (amount == 0) revert ZERO_AMOUNT();
74:     uint256 uBalanceBefore = uToken.balanceOf(address(this));
75:     uToken.safeTransferFrom(msg.sender, address(this), amount);
76:     uint256 uBalanceAfter = uToken.balanceOf(address(this));
77:
78:     uint256 cBalanceBefore = cToken.balanceOf(address(this));
79:     if (cToken.mint(uBalanceAfter - uBalanceBefore) != 0)
80:         revert LEND_FAILED(amount);
81:     uint256 cBalanceAfter = cToken.balanceOf(address(this));
82:
83:     shareAmount = cBalanceAfter - cBalanceBefore;
84:     _mint(msg.sender, shareAmount);
85:
86:     emit Deposited(msg.sender, amount, shareAmount);
87: }
```

## Tool used

Manual Review

## Recommendation

Consider calculating the delta of the token balance before and after the transfer and use this delta value as the `amount` to lend out.
