sayan_

medium

# gas limit DoS via unbounded loop

## Summary
Unbounded loop in accrueAll(),whitelistTokens(),whitelistSpells(),whitelistContracts() can lead to DoS
## Vulnerability Detail
If a very large array of token/spell/contract addresses is passed in the mentioned functions then the the loop inside has to iterate through all of that & perform all the instructions.With all of this happening in the loop and costing gas it may revert due to exceeding the block size gas limit. 

## Impact
Execution fails due to exceeding the block size gas limit
## Code Snippet
[BlueBerryBank.sol#L261-#L265](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L261-#L265)
```solidity
File: contracts/BlueBerryBank.sol
261:     function accrueAll(address[] memory tokens) external {
262:         for (uint256 idx = 0; idx < tokens.length; idx++) {
263:             accrue(tokens[idx]);
264:         }
265:     }
```
[BlueBerryBank.sol#L140-#L146](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L140-#L146)
```solidity
File: contracts/BlueBerryBank.sol
140:         for (uint256 idx = 0; idx < contracts.length; idx++) { 
141:             if (contracts[idx] == address(0)) {
142:                 revert ZERO_ADDRESS();
143:             }
144:             whitelistedContracts[contracts[idx]] = statuses[idx];
145:         }
```
[BlueBerryBank.sol#L158-#L164](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L158-#L164)
```solidity
File: contracts/BlueBerryBank.sol
158:         for (uint256 idx = 0; idx < spells.length; idx++) { 
159:             if (spells[idx] == address(0)) {
160:                 revert ZERO_ADDRESS();
161:             }
162:             whitelistedSpells[spells[idx]] = statuses[idx];
163:         }
```
[BlueBerryBank.sol#L176-#L181](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L176-#L181)
```solidity
File: contracts/BlueBerryBank.sol
176:         for (uint256 idx = 0; idx < tokens.length; idx++) {//@audit unbounded loop DoS
177:             if (statuses[idx] && !oracle.support(tokens[idx]))
178:                 revert ORACLE_NOT_SUPPORT(tokens[idx]);
179:             whitelistedTokens[tokens[idx]] = statuses[idx];
180:         }
```
## Tool used

Manual Review

## Recommendation
Consider avoiding all the actions executed in a single transaction, especially when calls are executed as part of a loop.