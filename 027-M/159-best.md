carrot

high

# Incorrect handling of token approvals

## Summary
The `BlueBerryBank` contract approves token spending. However, it never resets token approvals 0. This can cause issues with tokens like CRV, where you cannot set approvals from a non-zero value to another non-zero value.
## Vulnerability Detail
Approvals are generally handled in two ways. In the normal Openzeppelin pattern, approve function sets the allowance to the value passed. However in other tokens, like USDT, there is a built-in race condition prevention mechanism where approvals must be reset to 0 before setting it to some non-zero value. This is also true for the case of the CRV token, as can be seen from its vyper contract
```vyper
def approve(_spender : address, _value : uint256) -> bool:
    """
    @notice Approve `_spender` to transfer `_value` tokens on behalf of `msg.sender`
    @dev Approval may only be from zero -> nonzero or from nonzero -> zero in order
        to mitigate the potential race condition described here:
        https://github.com/ethereum/EIPs/issues/20#issuecomment-263524729
    @param _spender The address which will spend the funds
    @param _value The amount of tokens to be spent
    @return bool success
    """
    assert _value == 0 or self.allowances[msg.sender][_spender] == 0
    self.allowances[msg.sender][_spender] = _value
    log Approval(msg.sender, _spender, _value)
    return True
```

Since the protocol is intent on using CRV, it must use precaution with the approval mechanism and have the bank reset the approval to 0, after every instance of it granting approval. Otherwise, it can permanently brick the contract if even 1 wei of approval is remaining after calling the transfer functions, as it will revert on subsequent approval calls. Since the bank sometimes depends on external contracts to move out tokens (lending protocol, spells), the current approval pattern is very risky.

## Impact
Bricked bank for CRV tokens due to incorrect approval handling
## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L646-L657
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L882-L884
## Tool used

Manual Review

## Recommendation
Explicitly reset approval to 0 after setting it at the end of the functions
```solidity
IERC20Upgradeable(token).approve(bank.cToken, 0);
```