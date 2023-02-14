Bauer

medium

# Missing approve to zero could cause certain token transfer to fail

## Summary
In BlueBerry protocol there are functions ```lend()``` ,```withdrawLend()``` and ```doRepay()``` ... , which accepts USDC, DAI, ALCX, BAL, CRV, ICHI, SUSHI, WBTC, WETH in the whitelist. And the list may extend. But if token  dont support to change the allowance from an existing non-zero allowance value is added. Then the ``` approve()``` will revert if the current approval has not been set to zero.(e.g. USDT)
 

## Vulnerability Detail
```solidity
function approve(address _spender, uint _value) public onlyPayloadSize(2 * 32) {

        // To change the approve amount you first have to reduce the addresses`
        //  allowance to zero by calling `approve(_spender, 0)` if it is not
        //  already 0 to mitigate the race condition described here:
        //  https://github.com/ethereum/EIPs/issues/20#issuecomment-263524729
        require(!((_value != 0) && (allowed[msg.sender][_spender] != 0)));

        allowed[msg.sender][_spender] = _value;
        Approval(msg.sender, _spender, _value);
    }
```
As the code above , we find that if the address already has some USDT approved for contract and no reset beforeBefore the next operation ， the ``` approve()``` will revert .
Below functions using ```approve()``` or ```ensureApprove()``` should first be approved by zero. 

```solidity
 function lend(address token, uint256 amount)
        external
        override
        inExec
        poke(token)
        onlyWhitelistedToken(token)
    {
        if (!isLendAllowed()) revert LEND_NOT_ALLOWED();

        Position storage pos = positions[POSITION_ID];
        Bank storage bank = banks[token];
        if (pos.underlyingToken != address(0)) {
            // already have isolated collateral, allow same isolated collateral
            if (pos.underlyingToken != token)
                revert INCORRECT_UNDERLYING(token);
        }

        IERC20Upgradeable(token).safeTransferFrom(
            pos.owner,
            address(this),
            amount
        );
        amount = doCutDepositFee(token, amount);
        pos.underlyingToken = token;
        pos.underlyingAmount += amount;

        if (address(ISoftVault(bank.softVault).uToken()) == token) {
            IERC20Upgradeable(token).approve(bank.softVault, amount);
            pos.underlyingVaultShare += ISoftVault(bank.softVault).deposit(
                amount
            );
        } else {
            IERC20Upgradeable(token).approve(bank.hardVault, amount);
            pos.underlyingVaultShare += IHardVault(bank.hardVault).deposit(
                token,
                amount
            );
        }
 function withdrawLend(address token, uint256 shareAmount)
        external
        override
        inExec
        poke(token)
    {
        Position storage pos = positions[POSITION_ID];
        Bank storage bank = banks[token];
        if (token != pos.underlyingToken) revert INVALID_UTOKEN(token);
        if (shareAmount == type(uint256).max) {
            shareAmount = pos.underlyingVaultShare;
        }

        uint256 wAmount;
        if (address(ISoftVault(bank.softVault).uToken()) == token) {
            ISoftVault(bank.softVault).approve(
                bank.softVault,
                type(uint256).max
            );
 function doRepay(address token, uint256 amountCall)
        internal
        returns (uint256 repaidAmount)
    {
        Bank storage bank = banks[token]; // assume the input is already sanity checked.
        IERC20Upgradeable(token).approve(bank.cToken, amountCall);
        if (ICErc20(bank.cToken).repayBorrow(amountCall) != 0)
            revert REPAY_FAILED(amountCall);
        uint256 newDebt = ICErc20(bank.cToken).borrowBalanceStored(
            address(this)
        );
   function doRepay(address token, uint256 amount) internal {
        if (amount > 0) {
            ensureApprove(token, address(bank));
            bank.repay(token, amount);
        }
    }
   function doPutCollateral(address token, uint256 amount) internal {
        if (amount > 0) {
            ensureApprove(token, address(werc20));
            werc20.mint(token, amount);
            bank.putCollateral(
                address(werc20),
                uint256(uint160(token)),
                amount
            );
        }
    }
    function depositInternal(
        uint256 strategyId,
        address collToken,
        address borrowToken,
        uint256 collAmount,
        uint256 borrowAmount
    ) internal {
        Strategy memory strategy = strategies[strategyId];

        // 1. Lend isolated collaterals on compound
        doLend(collToken, collAmount);

        // 2. Borrow specific amounts
        doBorrow(borrowToken, borrowAmount);

        // 3. Add liquidity - Deposit on ICHI Vault
        IICHIVault vault = IICHIVault(strategy.vault);
        bool isTokenA = vault.token0() == borrowToken;
        uint256 balance = IERC20(borrowToken).balanceOf(address(this));
        ensureApprove(borrowToken, address(vault));
   function openPositionFarm(
        uint256 strategyId,
        address collToken,
        address borrowToken,
        uint256 collAmount,
        uint256 borrowAmount,
        uint256 farmingPid
    )
{
  ....
 // 5. Deposit on farming pool, put collateral
        ensureApprove(strategy.vault, address(wIchiFarm));
        uint256 lpAmount = IERC20(strategy.vault).balanceOf(address(this));
        uint256 id = wIchiFarm.mint(farmingPid, lpAmount);
        bank.putCollateral(address(wIchiFarm), id, lpAmount);
}

```

## Impact
The function ``` approve()``` will revert

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L620-L662
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L122-L157

## Tool used

Manual Review

## Recommendation
```solidity
e.g.
IERC20Upgradeable(token).approve(bank.softVault, 0);
IERC20Upgradeable(token).approve(bank.softVault, amount);
```
