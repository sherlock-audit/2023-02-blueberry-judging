0xbrett8571

high

# Incomplete Transfer in the Liquidate Function.

## Summary
The function calls the `safeTransferFrom` method of the `IERC1155` interface to transfer the collateral, but unfortunately, it does not check the return value of the call to ensure the transfer was successful, so as a result of such instance, if the transfer fails, the collateral may be lost, and the debt will remain unpaid.

## Vulnerability Detail
In the process of managing positions in debt, collateral, and underlying assets of a borrower. 
The `liquidate` function is meant to transfer the collateral when the debt is not paid. However, the function fails to check if the transfer was successful or not, which means that if the transfer fails, the collateral will be lost, and the debt remain unpaid.

In the `liquidate` function, the `safeTransferFrom` method of the `IERC1155` interface is called to transfer the collateral, but there is no return value check to ensure that the transfer was successful. The method returns a `boolean` value indicating whether the transfer was successful or not, but this value is not being used to confirm the transfer's success.

## Impact
If the transfer of collateral fails, the collateral will be permanently lost forever, and the debt will remain unpaid. This will result in financial loss for both parties involved and can harm the reputation of the smart contract platform and its ecosystem.

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L538-L544

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L511-L544

## Tool used

Manual Review

## Recommendation
My recommendation for the "liquidate". The `liquidate` function should check the return value of the call to `safeTransferFrom` method to ensure that the transfer of the collateral was successful.

Implementing this change will ensure that the collateral is not lost in case of a failed transfer, and the debt will not remain unpaid, it will revert with the error message "Transfer of the collateral failed." or "Transfer of the underlying collateral failed." in case of a failed transfer, which can be handled accordingly.

Here's one way to do that, but I am sure the dev has a more proper way.
```solidity
// Transfer position (Wrapped LP Tokens) to liquidator
+ bool transferSuccess = IERC1155Upgradeable(pos.collToken).safeTransferFrom(
    address(this),
    msg.sender,
    pos.collId,
    liqSize,
    ""
);

// Check if the transfer was successful
+ require(transferSuccess, "Transfer of the collateral failed.");

// Transfer underlying collaterals(vault share tokens) to liquidator
if (
    address(ISoftVault(bank.softVault).uToken()) == pos.underlyingToken
) {
+     transferSuccess = IERC20Upgradeable(bank.softVault).safeTransfer(
        msg.sender,
        uVaultShare
    );
} else {
+     transferSuccess = IERC1155Upgradeable(bank.hardVault).safeTransferFrom(
        address(this),
        msg.sender,
        uint256(uint160(pos.underlyingToken)),
        uVaultShare,
        ""
    );
}

// Check if the transfer was successful
+ require(transferSuccess, "Transfer of the underlying collateral failed.");
```