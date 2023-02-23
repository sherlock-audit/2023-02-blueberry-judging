koxuan

high

# user cannot add collateral to push position out of liquidatable status

## Summary
After clarification with the protocol team, a user should be able to add isolated collateral to push position out of liquidatable status. However, a `isLiquidatable` check in `execute` will prevent user from adding collateral in the event the position drops into liquidatable status.

## Vulnerability Detail

Increase position in `IchiVaultSpell` allows user to add isolated collateral to a position so that user can push position away from liquidatable status.


```solidity
    /**
     * @dev Increase isolated collateral of position
     * @param token Isolated collateral token address
     * @param amount Amount of token to increase position
     */
    function increasePosition(address token, uint256 amount) external {
        // 1. Get user input amounts
        doLend(token, amount);
    }


```
```solidity
    function doLend(address token, uint256 amount) internal {
        if (amount > 0) {
            bank.lend(token, amount);
        }
    }
```


Notice that  for `lend`, `inExec` is required, meaning that it must be a spell that is executed from `BlueBerryBank`.

```solidity
    /**
     * @dev Lend tokens to bank as isolated collateral. Must only be called while under execution.
     * @param token The token to deposit on bank as isolated collateral
     * @param amount The amount of tokens to lend.
     */
    function lend(address token, uint256 amount)
        external
        override
        inExec
        poke(token)
        onlyWhitelistedToken(token)
    {
        if (!isLendAllowed()) revert LEND_NOT_ALLOWED();

        Position storage pos = positions[POSITION_ID];
        Bank storage bank = banks[token];
        if (pos.underlyingToken != address(0)) {
            // already have isolated collateral, allow same isolated collateral
            if (pos.underlyingToken != token)
                revert INCORRECT_UNDERLYING(token);
        }

        IERC20Upgradeable(token).safeTransferFrom(
            pos.owner,
            address(this),
            amount
        );
        amount = doCutDepositFee(token, amount);
        pos.underlyingToken = token;
        pos.underlyingAmount += amount;

        if (address(ISoftVault(bank.softVault).uToken()) == token) {
            IERC20Upgradeable(token).approve(bank.softVault, amount);
            pos.underlyingVaultShare += ISoftVault(bank.softVault).deposit(
                amount
            );
        } else {
            IERC20Upgradeable(token).approve(bank.hardVault, amount);
            pos.underlyingVaultShare += IHardVault(bank.hardVault).deposit(
                token,
                amount
            );
        }

        bank.totalLend += amount;

        emit Lend(POSITION_ID, msg.sender, token, amount);
    }


```
See `if (isLiquidatable(positionId)) revert INSUFFICIENT_COLLATERAL();`, it checks if position is in liquidatable status. Once position reaches the liquidatable status, user cannot `increasePosition` to push position out of liquidatable status.

```solidity
    /// @dev Execute the action with the supplied data.
    /// @param positionId The position ID to execute the action, or zero for new position.
    /// @param spell The target spell to invoke the execution.
    /// @param data Extra data to pass to the target for the execution.
    function execute(
        uint256 positionId,
        address spell,
        bytes memory data
    ) external payable lock onlyEOAEx returns (uint256) {
        if (!whitelistedSpells[spell]) revert SPELL_NOT_WHITELISTED(spell);
        if (positionId == 0) {
            positionId = nextPositionId++;
            positions[positionId].owner = msg.sender;
        } else {
            if (positionId >= nextPositionId) revert BAD_POSITION(positionId);
            if (msg.sender != positions[positionId].owner)
                revert NOT_FROM_OWNER(positionId, msg.sender);
        }
        POSITION_ID = positionId;
        SPELL = spell;

        (bool ok, bytes memory returndata) = SPELL.call{value: msg.value}(data);
        if (!ok) {
            if (returndata.length > 0) {
                assembly {
                    let returndata_size := mload(returndata)
                    revert(add(32, returndata), returndata_size)
                }
            } else {
                revert("bad cast call");
            }
        }

        if (isLiquidatable(positionId)) revert INSUFFICIENT_COLLATERAL();

        POSITION_ID = _NO_ID;
        SPELL = _NO_ADDRESS;

        return positionId;
    }

```

## Impact
User is unable to retain his position once it reaches liquidatable status and has to fight with other people to liquidate his position to get back his collateral.

## Code Snippet
[IchiVaultSpell.sol#L252-L259](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L252-L259)
[BlueBerryBank.sol#L81-L85](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/BasicSpell.sol#L81-L85)
[BlueBerryBank.sol#L620-L626](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L620-L626)
[BlueBerryBank.sol#L607](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L607)


## Tool used

Manual Review

## Recommendation

Recommend allowing `lend` to be called even if `isLiquidatable` is true.
