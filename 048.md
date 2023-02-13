koxuan

medium

# no refund implemented in execute might lead to caller losing excess native tokens

## Summary
According to the docs and source code, `execute` is `payable` to allow longing ethereum strategies. However, upon looking into the code, there is no refund to user in the event that excess ethereum is returned to `BlueBerryBank`, causing excess ethereum to be stuck in the bank and loss of fund to caller.

## Vulnerability Detail
In `BlueBerryBank`, `execute` is called to execute a spell (strategy). Notice that the function is payable and according to the docs, it states longing ethereum as a strategy. Spells are called with `msg.value` sent over, in the event that the spell refund excess ethereum to the bank due to the data specifying a lesser amount than `msg.value`, the excess ethereum will be stuck in `BlueBerryBank` and hence loss of fund to user. 
```solidity
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
Excess ethereum will be stuck in `BlueBerryBank` in the event excess ethereum is refunded by spell to the bank.

## Code Snippet
[BlueBerryBank.sol#578-L613](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L578-L613)

## Tool used

Manual Review

## Recommendation

Recommend implementing  a refund of excess ethereum to caller. 
