tsvetanovv

medium

# Missing the feature to remove an user from whitelist

## Summary
[BlueBerryBank.sol](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L133) has a whitelist mechanism, but no function to remove the whitelist user. 

## Vulnerability Detail
The protocol is missing the feature to remove an user from whitelist. Once an user has been added to the whitelist, it is not possible to remove him from the whitelist. This applies not only to a malicious/honest user, but also to owner error. If the owner accidentally whitelisted the wrong user, there is no way to remove it.

## Impact
It is also possible a honest whitelisted user is found to be vulnerable and has been actively exploited by an attacker and the protocol needs to mitigate the issue swiftly by removing the vulnerable user from the protocol. 

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L133
```solidity
function whitelistContracts(  
        address[] calldata contracts,
        bool[] calldata statuses
    ) external onlyOwner {
        if (contracts.length != statuses.length) {
            revert INPUT_ARRAY_MISMATCH();
        }
        for (uint256 idx = 0; idx < contracts.length; idx++) {
            if (contracts[idx] == address(0)) {
                revert ZERO_ADDRESS();
            }
            whitelistedContracts[contracts[idx]] = statuses[idx];
        }
    }
```

## Tool used

Manual Review

## Recommendation
Consider implementing an additional function to allow the removal of an user from the whitelist.