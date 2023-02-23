0xmuxyz

high

# Once the owner change the status of tokens whitelisted, a user may lose their funds that was deposited as a collateral or the user's funds that was deposited as a collateral will be frozen within the Bank

## Summary
Once the owner change the status of tokens whitelisted, a user may lose their funds that was deposited as a collateral or the user's funds that was deposited as a collateral will be frozen within the Bank.

## Vulnerability Detail
Within the BlueBerryBank, the mapping storage of the `whitelistedTokens` is defined like this:
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L46
```solidity
    mapping(address => bool) public whitelistedTokens; // Mapping from token to whitelist status
```

Within the BlueBerryBank, 
the BlueBerryBank# `whitelistTokens()` is defined in order for new `token` to be added to the `whitelistedTokens` like this:
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L169-L181
```solidity
    /// @dev Set whitelist token status
    /// @param tokens list of tokens to change status
    /// @param statuses list of statuses to change to
    function whitelistTokens(
        address[] calldata tokens,
        bool[] calldata statuses
    ) external onlyOwner {
        if (tokens.length != statuses.length) {
            revert INPUT_ARRAY_MISMATCH();
        }
        for (uint256 idx = 0; idx < tokens.length; idx++) {
            if (statuses[idx] && !oracle.support(tokens[idx]))
                revert ORACLE_NOT_SUPPORT(tokens[idx]);
            whitelistedTokens[tokens[idx]] = statuses[idx];
        }
    }
```

Within the BlueBerryBank, 
the `onlyWhitelistedToken()` modifier to check whether or not the `token` assigned is whitelisted by using the `whitelistedTokens` is defined like this:
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L62-L65
```solidity
    /// @dev Ensure that the token is already whitelisted
    modifier onlyWhitelistedToken(address token) {
        if (!whitelistedTokens[token]) revert TOKEN_NOT_WHITELISTED(token);
        _;
    }
```

The `onlyWhitelistedToken()` modifier is used on the BlueBerryBank# `borrow()` and the BlueBerryBank# `repay()` in order to check whether or not the `token` assigned is whitelisted like this:
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L714
```solidity
    function borrow(address token, uint256 amount)
        external
        override
        inExec
        poke(token)
        onlyWhitelistedToken(token)  /// @audit
    {
        ...
```
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L745
```solidity
    function repay(address token, uint256 amountCall)
        external
        override
        inExec
        poke(token)
        onlyWhitelistedToken(token)  /// @audit
    {
        ...
```

If the owner would like to change the status of some token or remove some token from the `whitelistedTokens`, the owner can change it or remove it by calling the BlueBerryBank# `whitelistTokens()` again.

However, the certain scenarios like this would not be considered:
Scenario①). 
1. The owner set the initial whitelist token status by calling the BlueBerryBank# `whitelistTokens()`. Let's say the tokenA and tokenB are whitelisted (the status of tokenA is `true`) at this time.
2. A user deposits some amount of the tokenA whitelisted as a collateral to open a position and borrow debt tokenB.
3. The owner change the status of tokenB to `false` by calling the BlueBerryBank# `whitelistTokens()` again.
4. The user cannot repay the debt tokenB, meaning that the user lose the opportunity to repay the debt tokenB unless the owner change the status of tokenB to `true` again (by calling the BlueBerryBank# `whitelistTokens()` again).
5. If the price of tokenA that is deposited as a collateral suddenly goes down due to the change of huge market condition before the owner change the status of tokenB to `true` again, the amount of tokenA that is deposited by the user will be liquidated. 
In this case, the user lose their funds (tokenA) that was deposited as a collateral.

Scenario②).
1. The owner set the initial whitelist token status by calling the BlueBerryBank# `whitelistTokens()`. Let's say the tokenA and tokenB are whitelisted (the status of tokenA is `true`) at this time.
2. A user deposits some amount of the tokenA whitelisted as a collateral to open a position and borrow debt tokenB.
3. The owner change the status of tokenA to `false` by calling the BlueBerryBank# `whitelistTokens()` again.
4. The user cannot withdraw the tokenA even if the user repay full amount of debt tokenB.
In this case, the user's funds (tokenA) that was deposited as a collateral will be frozen within the Bank unless the owner change the status of tokenB to `true` again (by calling the BlueBerryBank# `whitelistTokens()` again).


## Impact
In case of the certain scenario,
- A user may lose their funds that was deposited as a collateral.
- The user's funds that was deposited as a collateral will be frozen within the Bank.

## Code Snippet
- https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L46
- https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L169-L181
- https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L62-L65
- https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L714
- https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L745

## Tool used
Manual Review

## Recommendation
Consider adding a validation to the BlueBerryBank# `whitelistTokens()` in order to check whether or not the positions opened that the tokens are used as a collateral token or debt token would be still remained, which the tokens are currently whitelisted but the status of it will be changed to `false` after the owner changes the status of tokens whitelisted (by calling the BlueBerryBank# `whitelistTokens()` again).

