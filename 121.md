Saeedalipoor01988

high

# The use of the platform can be disrupted if access to Chainlink oracle data feed is blocked

## Summary
Based on the current implementation, for getting a collateral token's price, the protocol should use Chainlink oracle data feed. but As https://blog.openzeppelin.com/secure-smart-contract-guidelines-the-dangers-of-price-oracles/ mentions, it is possible that Chainlink’s "multisigs can immediately block access to price feeds at will". 

## Vulnerability Detail
Based on the current implementation, for getting a collateral token's price, the protocol should use Chainlink oracle data feed. but As https://blog.openzeppelin.com/secure-smart-contract-guidelines-the-dangers-of-price-oracles/ mentions, it is possible that Chainlink’s "multisigs can immediately block access to price feeds at will". 
When this occurs, executing feeds[token].feed.latestAnswer() will revert so calling the viewPrice and getPrice functions also revert, which causes denial of service.

## Impact
the contract will call the isLiquidatable function in the BlueBerryBank contract to check whether the position is Liquidatable or not! isLiquidatable will call the getPositionRisk function and in this function, we need getPositionValue and getDebtValue from the CoreOracle contract. to get getPositionValue and getDebtValue we need to get the price from the Chainlink oracle data feed.

but As https://blog.openzeppelin.com/secure-smart-contract-guidelines-the-dangers-of-price-oracles/ mentions, it is possible that Chainlink’s "multisigs can immediately block access to price feeds at will".  When this occurs, executing feeds[token].feed.latestAnswer() will revert so calling the viewPrice and getPrice functions also revert, which causes denial of service in getPositionValue and getDebtValue functions.

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/CoreOracle.sol#L143
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/CoreOracle.sol#L164
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/ChainlinkAdapterOracle.sol#L66

## Tool used

Manual Review

## Recommendation
The logic for getting the collateral token's price from the Chainlink oracle data feed should be placed in the try block while some fallback logic when the access to the Chainlink oracle data feed is denied should be placed in the catch block. If getting the fixed price for the collateral token is considered as a fallback logic, then setting the fixed price for the token should become mandatory, which is different from the current implementation. Otherwise, fallback logic for getting the token's price from a fallback oracle is needed.