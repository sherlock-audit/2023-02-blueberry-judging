mgf15

medium

# initialize function miss modifier !

## Summary
initialize function on [BlueBerryBank.sol#L98](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L94) is a public function with no modifier and miss owner check the first one call it will own the contract ! 
## Vulnerability Detail
after deploy BlueBerryBank.sol  initialize function must be called to initialize but it's public function and any one can call it , the first one call it will own the contract .
## Impact
Loss in Access Control
## Code Snippet
[BlueBerryBank.sol#L98](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L94) 
```solidity
 function initialize(IOracle _oracle, IProtocolConfig _config)
        external
        initializer
    {
        __Ownable_init();
        if (address(_oracle) == address(0) || address(_config) == address(0)) {
            revert ZERO_ADDRESS();
        }
        _GENERAL_LOCK = _NOT_ENTERED;
        _IN_EXEC_LOCK = _NOT_ENTERED;
        POSITION_ID = _NO_ID;
        SPELL = _NO_ADDRESS;

        config = _config;
        oracle = _oracle;
        nextPositionId = 1;
        bankStatus = 7; // allow borrow, lend, repay

        emit SetOracle(address(_oracle));
    }
```
## Tool used

Manual Review

## Recommendation
add a require to check who call it for initialize