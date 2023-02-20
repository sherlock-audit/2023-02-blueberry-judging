PRAISE

high

# ISOLATED COLLATERALS CAN BE STOLEN  IN ICHIVAULTSPELL.SOL VIA reducePosition() AS THERE IS NO ACCESS CONTROL IMPLEMENTED

## Summary
ISOLATED COLLATERALS CAN BE STOLEN AS THERE IS NO ACCESS CONTROL IMPLEMENTED.

## Vulnerability Detail
Due to lack of  access control implementation anyone can make a call to reducePosition() function as it is external.


## Impact
The reducePosition() function calls BasicSpell's doRefund() which has a safeTransfer and calls  iBank's Executor() which returns an address.
**This can be used to withdraw the collateral of another user( ** via doRefund() ** ) by inputting the address of the user's collateral**. 

**There is no check to make sure that a user can't withdraw another user's collateral VIA reducePosition() which calls doRefund()**, i.e, A USER SHOULD BE ABLE TO REDUCE ONLY HIS/HER POSITION AND NOT THE POSITION OF ANOTHER USER.

ALEX CAN WITHDRAW ALICE'S COLLATERAL VIA reducePosition() which calls doRefund(), BECAUSE THERE IS NO CHECK IN PLACE TO MAKE SURE _function reducePosition()_ CAN ONLY BE USED TO REDUCE THE POSITION OF ONE'S POSITION AND NOT THE POSITION OF ANOTHER USER.

Also the supposed check for strategyid is called only after doRefund() is called.
```solidity
  doRefund(collToken);
  _validateMaxLTV(strategyId);
```
## Code Snippet
 https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L266-L274
```solidity
    function doRefund(address token) internal {
        uint256 balance = IERC20Upgradeable(token).balanceOf(address(this));
        if (balance > 0) {
            IERC20Upgradeable(token).safeTransfer(bank.EXECUTOR(), balance);
        }
    }
```
```solidity
    function EXECUTOR() external view returns (address);
```
## Tool used

Manual Review

## Recommendation 
implement Access control on the reducePosition() external function, i.e put a check to make sure that it's only the owner of the isolated collateral that can reduce the position.