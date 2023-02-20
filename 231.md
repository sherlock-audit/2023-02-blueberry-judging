eierina

medium

# Incomplete / incorrect ERC-165 interface support

## Summary


[ERC1155NaiveReceiver](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/utils/ERC1155NaiveReceiver.sol) is missing ERC-165 interface support.

## Vulnerability Detail

The ERC1155NaiveReceiver implements ERC-165 incorrectly, so that a component that correctly implements ERC-1155 Token Receiver support check would fail to detect such interface on a ERC1155NaiveReceiver derived contract. In such case, the component would query ERC-165 support first (`interfaceId == type(IERC165).interfaceId`) and fail, therefore not even trying to check for ERC-1155 Token Receiver support (`interfaceId == type(IERC1155Receiver).interfaceId`).
 
```solidity
contract ERC1155NaiveReceiver is IERC1155Receiver {

    ...

    function supportsInterface(
        bytes4 interfaceId
    ) external view virtual override returns (bool) {
        return interfaceId == type(IERC1155Receiver).interfaceId;
    }
}
```

In [ERC-1155: Multi Token Standard](https://eips.ethereum.org/EIPS/eip-1155), ERC-165 is explicitly mentioned in the [ERC-1155 Token Receiver](https://eips.ethereum.org/EIPS/eip-1155#erc-1155-token-receiver) section, and it is stated that:
>Smart contracts MUST implement all of the functions in the ERC1155TokenReceiver interface to accept transfers. See “Safe Transfer Rules” for further detail.
> Smart contracts MUST implement the ERC-165 supportsInterface function and signify support for the ERC1155TokenReceiver interface to accept transfers. See “ERC1155TokenReceiver ERC-165 rules” for further detail.

The `ERC1155TokenReceiver ERC-165 rules` then states:

> - The implementation of the ERC-165 supportsInterface function SHOULD be as follows:
> ```solidity
>   function supportsInterface(bytes4 interfaceID) external view returns (bool) {
>       return  interfaceID == 0x01ffc9a7 ||    // ERC-165 support (i.e. `bytes4(keccak256('supportsInterface(bytes4)'))`).
>               interfaceID == 0x4e2312e0;      // ERC-1155 `ERC1155TokenReceiver` support (i.e. `bytes4(keccak256("onERC1155Received(address,address,uint256,uint256,bytes)")) ^ bytes4(keccak256("onERC1155BatchReceived(address,address,uint256[],uint256[],bytes)"))`).
>   }
> ```
> - The implementation MAY differ from the above but:
>   - It MUST return the constant value true if 0x01ffc9a7 is passed through the interfaceID argument. This signifies ERC-165 support.
>   - It MUST return the constant value true if 0x4e2312e0 is passed through the interfaceID argument. This signifies ERC-1155 ERC1155TokenReceiver support.
>   - It MUST NOT consume more than 10,000 gas.
>     - This keeps it below the ERC-165 requirement of 30,000 gas, reduces the gas reserve needs and minimises possible side-effects of gas exhaustion during the call.

in [ERC-165: Standard Interface Definition](https://eips.ethereum.org/EIPS/eip-165#how-to-detect-if-a-contract-implements-erc-165) it is explained [how to detect if a contract implements any given interface](https://eips.ethereum.org/EIPS/eip-165#how-to-detect-if-a-contract-implements-any-given-interface):

> ### How to Detect if a Contract Implements ERC-165
> 
> 1. The source contract makes a STATICCALL to the destination address with input data: 0x01ffc9a701ffc9a700000000000000000000000000000000000000000000000000000000 and gas 30,000. This corresponds to contract.supportsInterface(0x01ffc9a7).
> 2. If the call fails or return false, the destination contract does not implement ERC-165.
> 3. If the call returns true, a second call is made with input data 0x01ffc9a7ffffffff00000000000000000000000000000000000000000000000000000000.
> 4. If the second call fails or returns true, the destination contract does not implement ERC-165.
> 5. Otherwise it implements ERC-165.
> 
> ### How to Detect if a Contract Implements any Given Interface
> 
> 1. If you are not sure if the contract implements ERC-165, use the above procedure to confirm.
> 2. If it does not implement ERC-165, then you will have to see what methods it uses the old-fashioned way.
> 3. If it implements ERC-165 then just call supportsInterface(interfaceID) to determine if it implements an interface you can use.

## Impact
Other protocols would fail to detect ERC-1155 Token Receiver support and therefore fail/refuse to transfer tokens to an ERC1155NaiveReceiver based contract.

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/utils/ERC1155NaiveReceiver.sol#L30-L34

## Tool used

Manual Review

## Recommendation

Add missing ERC-165 support code.

```solidity
pragma solidity 0.8.16;

import "@openzeppelin/contracts/token/ERC1155/IERC1155Receiver.sol";
import "@openzeppelin/contracts/utils/introspection/ERC165.sol";

abstract contract ERC1155NaiveReceiver is ERC165, IERC1155Receiver {
    uint256[49] private __gap;

    function onERC1155Received ...
    function onERC1155BatchReceived ...

    function supportsInterface(bytes4 interfaceId) public view virtual override(ERC165, IERC165) returns (bool) {
        return interfaceId == type(IERC1155Receiver).interfaceId || super.supportsInterface(interfaceId);
    }
}
```