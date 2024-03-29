Cryptor

medium

# Liquidate does not update the owner of the new position to the msg.sender after it has been completed

## Summary
The function Liquidate does not update the owner after the position has been liquidated 


## Vulnerability Detail
The function Liquidate can be called by anyone. According to the docs this is intended and not a flaw in the design of the protocol. After the msg.sender pays the debt for the owner, the function transfers both the position (ERC1155 token) shown here

 https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L537-L544

  as well as the collateral (vault shares) to the msg.sender as shown here https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L545-L560. 

However, there is no functionality that updates the owner of the struct position to be the liquidator after the position has been liquidated. Therefore the original owner of the position never changes. The docs mention that a bot will auto liquidate a position once it passes the threshold, however, if someone front runs the bot, that person will not be able to interact with the protocol properly with the newly owned collateral and vault shares.


## Impact
Functions lend, getPositionInfo and function getPositionIdsByOwner will either not work properly or return the wrong information because they all reference the owner of the struct position which incorrectly does not change after the position has been liquidated.

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/interfaces/IBank.sol#L20-L30

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L315-L333


https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L335-L361



## Tool used

Manual Review

## Recommendation
After the position has been liquidated, update the owner to the msg.sender e.g. pos.owner = msg.sender
