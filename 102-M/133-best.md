koxuan

medium

# position owner can execute spell on position even if risk level has passed the liquidatable threshold.

## Summary
When executing spell, position will be checked to ensure that it is not in the liquidatable status. However, due to the design of the protocol to cache the `totalDebt` instead of querying compound forked protocol for the `totalDebt` everytime, cached `totalDebt` will not have the accrued interest from the point `execute` is called to the last `accrue` being called. User position might have entered liquidatable threshold but the cached `totalDebt` is not the latest and hence user can execute spell on position. 

## Vulnerability Detail

When user execute a spell on an existing position, a `isLiquiditable` check is run to make sure that user position has sufficient collateral.

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

In `isLiquidatable`, position risk is used to determine if position has entered liquidatable status.
```solidity
    function isLiquidatable(uint256 positionId)
        public
        view
        returns (bool liquidatable)
    {
        Position storage pos = positions[positionId];
        uint256 risk = getPositionRisk(positionId);
        liquidatable = risk >= oracle.getLiqThreshold(pos.underlyingToken);
    }
```

```solidity
    function getPositionRisk(uint256 positionId)
        public
        view
        returns (uint256 risk)
    {
        Position storage pos = positions[positionId];
        uint256 pv = getPositionValue(positionId);
        uint256 ov = getDebtValue(positionId);
        uint256 cv = oracle.getUnderlyingValue(
            pos.underlyingToken,
            pos.underlyingAmount
        );


        if (cv == 0) risk = 0;
        else if (pv >= ov) risk = 0;
        else {
            risk = ((ov - pv) * DENOMINATOR) / cv;
        }
    }
```
`getDebtValue` in `getPositionRisk` is part of the formula to calculate the risk. Look at `  uint256 debt = (share*bank.totalDebt).divCeil(bank.totalShare);` Notice that `bank.totalDebt`, the cached totalDebt of the bank is used to calculate position debt. 
 
```solidity
    function getDebtValue(uint256 positionId)
        public
        view
        override
        returns (uint256)
    {
        uint256 value = 0;
        Position storage pos = positions[positionId];
        uint256 bitMap = pos.debtMap;
        uint256 idx = 0;
        while (bitMap > 0) {
            if ((bitMap & 1) != 0) {
                address token = allBanks[idx];
                uint256 share = pos.debtShareOf[token];
                Bank storage bank = banks[token];
                uint256 debt = (share * bank.totalDebt).divCeil(
                    bank.totalShare
                );
                value += oracle.getDebtValue(token, debt);
            }
            idx++;
            bitMap >>= 1;
        }
        return value;
    }
```

If `accrue` is not called for a very long time due to inactivity for the token which means `poke` and `accrue` not being called, position debt will be lesser than the actual amount and thus owner can execute spell even if the current debt has passed the liquidatable threshold due to stale `totalDebt`.

Note:  Other than `totalDebtValue`, upstream contracts that call view only functions like `getBankInfo`, `borrowBalanceStored` and `getPositionDebts` will get a lesser debt amount. If there are frontends or external contracts that rely on these functions, returning a stale value will cause upstream interfaces that rely on it to not work correctly. 
   
## Impact
Position owner can execute spell on position even if risk level has passed the liquidatable threshold. 

## Code Snippet
[BlueBerryBank.sol#L578-L613](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L578-L613)
[BlueBerryBank.sol#L497-L505](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L497-L505)
[BlueBerryBank.sol#L477-L495](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L477-L495)
[BlueBerryBank.sol#L451-L475](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L451-L475)
## Tool used

Manual Review

## Recommendation

Recommend calling compound forked protocol directly for the `totalDebt` if  view only is needed for `getDebtValue` because compound protocol `borrowBalanceCurrent` is a view function unlike  `poke` and `accrue`.
