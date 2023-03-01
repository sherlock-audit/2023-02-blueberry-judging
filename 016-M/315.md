peanuts

medium

# 0 value transfer for fees should not be allowed

## Summary

0 value transfer for fees should not be allowed.

## Vulnerability Detail

In BlueBerryBank.sol, when token is lend to the bank as isolated collateral, a deposit fee is taken. `lend()` calls `doCutDepositFee()`

```solidity
        IERC20Upgradeable(token).safeTransferFrom(
            pos.owner,
            address(this),
            amount
        );
        amount = doCutDepositFee(token, amount);
```

`doCutDepositFee()` takes the `depositFee` amount from the ProtocolConfig contract and subsequently transfers the fee to the treasury.

```solidity
    function doCutDepositFee(address token, uint256 amount)
        internal
        returns (uint256)
    {
        if (config.treasury() == address(0)) revert NO_TREASURY_SET();
        uint256 fee = (amount * config.depositFee()) / DENOMINATOR;
        IERC20Upgradeable(token).safeTransfer(config.treasury(), fee);
        return amount - fee;
    }
```

In ProtocolConfig.sol, owner can set a deposit / withdrawal fee of up to 20%. This means that the owner can set a 0% fee for certain events (to lure people to the protocol). 

```solidity
    function setDepositFee(uint256 depositFee_) external onlyOwner {
        // Cap to 20%
        if (depositFee_ > 2000) revert FEE_TOO_HIGH(depositFee_);
        depositFee = depositFee_;
    }


    function setWithdrawFee(uint256 withdrawFee_) external onlyOwner {
        // Cap to 20%
        if (withdrawFee_ > 2000) revert FEE_TOO_HIGH(withdrawFee_);
        withdrawFee = withdrawFee_;
    }
```

If deposit fee is set to 0, the fee will be 0 and 0 value will be transferred to the treasury.

```solidity
        uint256 fee = (amount * config.depositFee()) / DENOMINATOR;
        IERC20Upgradeable(token).safeTransfer(config.treasury(), fee);
```

A similar issue can be found here: https://github.com/code-423n4/2022-02-aave-lens-findings/issues/62

Since [BlueBerry](https://docs.blueberry.garden/earn/listing) does accept tokens through whitelisting, there may be a case where the token does not accept zero value transfer. 

## Impact

Those functions that require fees will revert, such as `lend()` and `withdrawLend()`

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L637-L644

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L892-L900

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/ProtocolConfig.sol#L50-L60

## Tool used

Manual Review

## Recommendation

Only accept > 0 fees.

```solidity
    function doCutDepositFee(address token, uint256 amount)
        internal
        returns (uint256)
    {
        if (config.treasury() == address(0)) revert NO_TREASURY_SET();
        uint256 fee = (amount * config.depositFee()) / DENOMINATOR;
+      if(fee > 0){
        IERC20Upgradeable(token).safeTransfer(config.treasury(), fee);
+     }
        return amount - fee;
    }
```
