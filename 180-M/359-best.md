Avci

high

# Attacker can make users unable to withdraw their assets

## Summary
The withdraw function in the SoftVault contract doesn't validate that the contract has enough cTokens to redeem for the requested amount of shareAmount. This can cause the redeem function to fail, and the transaction will revert.

## Vulnerability Detail
The withdraw function in the Vault contract doesn't perform sufficient validation before redeeming cTokens for underlying assets. Specifically, the function doesn't check if the contract has enough cTokens to redeem for the requested amount of shareAmount. If the contract doesn't have enough cTokens, the redeem function will fail, and the transaction will revert after transferring tokens from the user.

## Impact
If an attacker exploits this vulnerability, they can create a situation where the contract doesn't have enough cTokens to redeem for the requested amount of shareAmount. As a result, the redeem function will fail, and the user's funds will be stuck in the contract. This could cause significant financial loss for the user. If this happens, the user's funds will not be lost, but they will be unable to withdraw their assets until the contract has obtained more cTokens.
This issue can cause inconvenience for users if they need to withdraw their funds urgently but are unable to do so because the contract doesn't have enough cTokens.

## Code Snippet
```solidity
 function withdraw(uint256 shareAmount) external override nonReentrant returns (uint256 withdrawAmount) {
    // ...
    if (cToken.redeem(shareAmount) != 0) revert REDEEM_FAILED(shareAmount);
    // ...
}
```

https://github.com/sherlock-audit/2023-02-blueberry-0xdanial/blob/55ebb9094b00c96ffffb27290fb305cd5d65ded6/contracts/vault/SoftVault.sol#L105
## Tool used

Manual Review

## Recommendation
simply checking the balance of cToken before calling redeem won't fully prevent this issue. To fully prevent this issue, the contract should use the IERC20.approve() function to give an allowance to the cToken contract before calling redeem(). This allows the cToken contract to transfer the necessary amount of uToken to the contract without the contract needing to hold any cToken balance.