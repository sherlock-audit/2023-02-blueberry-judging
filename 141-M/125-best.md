obront

medium

# setRoute function allows liqThreshold to be set to 0, bypassing checks

## Summary

While there are careful protections set up to ensure that the `liqThreshold` for all tokens added to the oracle falls into the acceptable range of 0.8 to 1.0, there is a loophole that allows tokens to be added with `liqThreshold = 0`, which could cause unfair liquidations for users.

## Vulnerability Detail

In `CoreOracle.sol#setTokenSettings()` there are careful checks to ensure that tokens can only be added with a liquidity threshold that is between 0.8 and 1.

```solidity
if (settings[idx].liqThreshold > DENOMINATOR)
    revert LIQ_THRESHOLD_TOO_HIGH(settings[idx].liqThreshold);
if (settings[idx].liqThreshold < MIN_LIQ_THRESHOLD)
    revert LIQ_THRESHOLD_TOO_LOW(settings[idx].liqThreshold);
```
This is very important, as if a liquidity threshold is ever accidentally set to 0, it would mean that all users who use that type of collateral are instantly liquidatable, since risk will always be greater than or equal to 0.
```solidity
function isLiquidatable(uint256 positionId)
    public
    view
    returns (bool liquidatable)
{
    ...
    liquidatable = risk >= oracle.getLiqThreshold(pos.underlyingToken);
}
```
Unfortunately, the `setRoute` function, which I believe is intended to be used to update a route, bypasses this check and allows new routes to be set up with a `route.liqThreshold = 0`.
```solidity
function setRoute(address[] calldata tokens, address[] calldata routes)
    external
    onlyOwner
{
    if (tokens.length != routes.length) revert INPUT_ARRAY_MISMATCH();
    for (uint256 idx = 0; idx < tokens.length; idx++) {
        if (tokens[idx] == address(0) || routes[idx] == address(0))
            revert ZERO_ADDRESS();

        tokenSettings[tokens[idx]].route = routes[idx];
        emit SetRoute(tokens[idx], routes[idx]);
    }
}
```
## Impact

Tokens can be set up in the oracle which pass all the validation and allow `liqThreshold` to be set to 0. In the event that this happens, any user who uses this form of collateral will become liquidatable.

I am aware that, typically, admin errors are exempt from Sherlock contests. However, it is clear that the protocol team has made an intentional effort to add the checks required to make this situation impossible, since it is an ongoing risk and not just a "one time setup". Additionally, an error would cause major harm to the user base. For that reason, I believe a Medium is justified.

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/CoreOracle.sol#L38-L50

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L497-L505

## Tool used

Manual Review

## Recommendation

Add a check to `setRoute` to ensure it is only called when the `liqThreshold` has already been set:
```diff
function setRoute(address[] calldata tokens, address[] calldata routes)
    external
    onlyOwner
{
    if (tokens.length != routes.length) revert INPUT_ARRAY_MISMATCH();
    for (uint256 idx = 0; idx < tokens.length; idx++) {
-        if (tokens[idx] == address(0) || routes[idx] == address(0)) 
+        if (tokens[idx] == address(0) || routes[idx] == address(0) || tokenSettings[tokens[idx]].liqThreshold == 0) 
            revert ZERO_ADDRESS();

        tokenSettings[tokens[idx]].route = routes[idx];
        emit SetRoute(tokens[idx], routes[idx]);
    }
}
```