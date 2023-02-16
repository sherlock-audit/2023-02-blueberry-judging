0xbrett8571

medium

# Use of public state variables in `whitelistedTokens`, `whitelistedSpells`, and `whitelistedContracts`.

## Summary
The state variables "whitelistedTokens", "whitelistedSpells", and "whitelistedContracts" in the contract are declared as public. By declaring these variables as public, anyone can access and modify their values, including bad actors and then use them to their advantage.

## Vulnerability Detail
The use of public state variables in the `whitelistedTokens`, `whitelistedSpells`, and `whitelistedContracts`. 
I see that these variables are declared as public, which means that their values can be directly accessed by any external contract or account.
```solidity
bool public allowContractCalls; // The boolean status whether to allow call from contract (false = onlyEOA)
mapping(address => bool) public whitelistedTokens; // Mapping from token to whitelist status
mapping(address => bool) public whitelistedSpells; // Mapping from spell to whitelist status
mapping(address => bool) public whitelistedContracts; // Mapping from user to whitelist status
```
But if by any chance a bad actor were to change the value of `whitelistedTokens`, they could cause the contract to process transactions to their advantage with tokens that it should not be processing. 
Additionally, if a malicious actor were to change the value of `whitelistedSpells`, they could cause the contract to process transactions from spells that it should not be processing, leading to security issues. 
Finally, if a malicious actor were to change the value of the `whitelistedContracts`, they could potentially cause the contract to process transactions from contracts that should not be processed, and cause functional issues.

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L45-L48
Lastly, I see that the contract does attempt to mitigate these risks with the `onlyEOAEx` and `onlyWhitelistedToken` modifiers, but these measures may not be enough to fully prevent malicious actors from manipulating these public state variables.

## Impact
Let's take for instance, if the `whitelistedTokens` mapping is manipulated by anyone, they can add unauthorized tokens to the list of approved tokens, allowing them to bypass security checks and perform actions that are unauthorized within the contract. The same is true for the `whitelistedSpells` and `whitelistedContracts` mappings. It can be exploited to cause significant financial loss to the contract's owner and users of the contract.

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L45-L48

## Tool used

Manual Review

## Recommendation
It will be best and better to switch to using `getter` functions instead, this way bad actors will not be able to directly manipulate the data stored in the state variables, and all access to the variables must go through the `getter` functions.

For example, here the `whitelistedTokens` mapping can be replaced with a getter function `isTokenWhitelisted(address token)` that returns the "whitelist" status of the given token, so the getter function can perform the necessary access control checks to ensure that only authorized parties are able to access the information.

I will also recommend adding access control modifiers `(e.g., onlyOwner, onlyWhitelisted)` to the getter functions to further restrict access to sensitive information.