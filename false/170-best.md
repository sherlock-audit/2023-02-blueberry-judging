foxb868

high

# "TransferFrom" calls without allowance checks.

## Summary
`lend` function uses the `IERC20Upgradeable.safeTransferFrom` method to transfer tokens, but it does not check if the user has approved the contract to spend the required amount of tokens.

## Vulnerability Detail
```solidity
 IERC20Upgradeable(token).safeTransferFrom(
            pos.owner,
            address(this),
            amount
        );
```
The function call transfers tokens from the `pos.owner` address to the contract address `(address(this))`, but it does not check if the user has approved the contract to spend the required amount of tokens. This means that an attacker could call the `lend` function to transfer tokens from the victim's account without their consent, as long as the contract holds "sufficient" balance of the same token.

In this case, the `lend` function is calling the `safeTransferFrom` method to transfer tokens without first checking if the user has already approved the contract to spend the required amount of tokens. This means that if a user has not granted approval, the transfer will fail, and the funds will not be transferred.

## Impact
It can allow someone with bad intent to drain a user's account by transferring a large `amount` of tokens without the user's approval.

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L637-L641

## Tool used

Manual Review & Vs Code.

## Recommendation
Add an allowance check before transferring tokens in the `lend` function.
By adding a check to ensure that the user has approved the contract to spend the required amount of tokens before executing the `safeTransferFrom` method. 

You can do it by using the `allowance` function of the ERC20 token interface to verify that the contract has been approved to spend the required amount of tokens.

Use the `allowance` function to get the current allowance of the user for the contract address, if the allowance is less than the required amount, the function should revert with `INSUFFICIENT_ALLOWANCE` error. If the allowance is sufficient, the function continues with the transfer of tokens as before.

```solidity
function lend(address token, uint256 amount) external override inExec poke(token) onlyWhitelistedToken(token) {
    if (!isLendAllowed()) revert LEND_NOT_ALLOWED();

    Position storage pos = positions[POSITION_ID];
    Bank storage bank = banks[token];
    if (pos.underlyingToken != address(0)) {
        // already have isolated collateral, allow same isolated collateral
        if (pos.underlyingToken != token)
            revert INCORRECT_UNDERLYING(token);
    }

    uint256 allowance = IERC20Upgradeable(token).allowance(pos.owner, address(this));
    if (allowance < amount) revert INSUFFICIENT_ALLOWANCE();

    IERC20Upgradeable(token).safeTransferFrom(pos.owner, address(this), amount);
    amount = doCutDepositFee(token, amount);
    pos.underlyingToken = token;
    pos.underlyingAmount += amount;

    if (address(ISoftVault(bank.softVault).uToken()) == token) {
        IERC20Upgradeable(token).approve(bank.softVault, amount);
        pos.underlyingVaultShare += ISoftVault(bank.softVault).deposit(amount);
    } else {
        IERC20Upgradeable(token).approve(bank.hardVault, amount);
        pos.underlyingVaultShare += bank.hardVault.deposit{value: 0}(uint256(uint160(token)), amount, msg.sender);
    }

    emit Lend(POSITION_ID, token, amount);
}
```