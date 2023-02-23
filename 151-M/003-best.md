seeu

medium

# ERC165Checker unbounded gas consumption

## Summary

ERC165Checker unbounded gas consumption

## Vulnerability Detail

According to the issue raised in [`CVE-2022-35915`](https://github.com/OpenZeppelin/openzeppelin-contracts/security/advisories/GHSA-7grf-83vw-6f5x):

> The target contract of an EIP-165 `supportsInterface` query can cause unbounded gas consumption by returning a lot of data, while it is generally assumed that this operation has a bounded cost.

## Impact

unbounded gas consumption

## Code Snippet

[package.json#L19](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/package.json#L19)
```json
"@openzeppelin/contracts": "^4.6.0",
```

[contracts/utils/ERC1155NaiveReceiver.sol#L30-L34](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/utils/ERC1155NaiveReceiver.sol#L30-L34)
```Solidity
function supportsInterface(
    bytes4 interfaceId
) external view virtual override returns (bool) {
    return interfaceId == type(IERC1155Receiver).interfaceId;
}
```

## Tool used

Manual Review

## Recommendation

It is highly suggested to update @openzeppelin/contracts to the [most recent version, 4.8.1](https://www.npmjs.com/package/@openzeppelin/contracts), or at least to the version 4.7.2 since the issue highlighted is patched from this version.