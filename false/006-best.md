seeu

medium

# NFT can be frozen in the contract

## Summary

NFT can be frozen in the contract

## Vulnerability Detail

The NFT can be frozen in the contract if `msg.sender` is an address for a contract that does not support ERC721.

References:
- [EIP-721](https://eips.ethereum.org/EIPS/eip-721)
- [ERC721.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC721/ERC721.sol#L274-L285)

## Impact

Users could lose their NFT.

## Code Snippet

- [contracts/mock/MockERC20.sol#L23](https://github.com/sherlock-audit/2023-02-blueberry-seeu-inspace/tree/main/contracts/mock/MockERC20.sol#L23)
  ```Solidity
  _mint(msg.sender, 100 * 10**_decimals);
  ```
- [contracts/mock/MockIchiV2.sol#L36](https://github.com/sherlock-audit/2023-02-blueberry-seeu-inspace/tree/main/contracts/mock/MockIchiV2.sol#L36)
  ```Solidity
  _mint(msg.sender, v2Amount);
  ```
- [contracts/vault/HardVault.sol#L81](https://github.com/sherlock-audit/2023-02-blueberry-seeu-inspace/tree/main/contracts/vault/HardVault.sol#L81)
  ```Solidity
  _mint(msg.sender, uint256(uint160(token)), shareAmount, "");
  ```
- [contracts/vault/SoftVault.sol#L84](https://github.com/sherlock-audit/2023-02-blueberry-seeu-inspace/tree/main/contracts/vault/SoftVault.sol#L84)
  ```Solidity
  _mint(msg.sender, shareAmount);
  ```
- [contracts/wrapper/WIchiFarm.sol#L108](https://github.com/sherlock-audit/2023-02-blueberry-seeu-inspace/tree/main/contracts/wrapper/WIchiFarm.sol#L108)
  ```Solidity
  _mint(msg.sender, id, amount, "");
  ```

## Tool used

Manual Review

## Recommendation

It is reccomended to use `_safeMint` instead of `_mint`.