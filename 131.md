0xChinedu

false

# Unhandled Chainlink revert Would Lock Access To Oracle Price Access

## Summary
Chainlink's  **latestRoundData()**  is used which could potentially revert and make it impossible to query any prices. This could lead to permanent denial of service.
## Vulnerability Detail
The **ChainlinkAdapterOracle.getPrice()** function makes use of Chainlink's  **latestRoundData()** to get the latest price. However, there is no fallback logic to be executed when the access to the Chainlink data feed is denied by Chainlink's multisigs. While currently there’s no whitelisting mechanism to allow or disallow contracts from reading prices, powerful multisigs can tighten these access controls. In other words, the multisigs can immediately block access to price feeds at will.
[https://blog.openzeppelin.com/secure-smart-contract-guidelines-the-dangers-of-price-oracles/](url)
## Impact
ChainlinkAdapterOracle.getPrice() could revert and cause denial of service to the protocol.
## Code Snippet
[https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/ChainlinkAdapterOracle.sol#L66-L84](url)

## Tool used

Manual Review

## Recommendation
Use **try/catch** block. The logic for getting the token's price from the Chainlink data feed should be placed in the **try** block, while some fallback logic when the access to the chainlink oracle data feed is denied should be placed in the **catch** block. E.g:
```solidity
function getPrice(address priceFeedAddress) external view returns (int256) {
        try AggregatorV3Interface(priceFeedAddress).latestRoundData() returns (
            uint80,         // roundID
            int256 price,   // price
            uint256,        // startedAt
            uint256,        // timestamp
            uint80          // answeredInRound
        ) {
            return price;
        } catch Error(string memory) {            
            // handle failure here:
            // revert, call propietary fallback oracle, fetch from another 3rd-party oracle, etc.
        }
    }
```