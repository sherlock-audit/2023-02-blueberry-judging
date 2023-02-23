RaymondFam

medium

# Use of deprecated `safeApprove()` is discouraged and missing approve to zero could cause certain token transfer to fail

## Summary
OpenZeppelin's `safeapprove()` has issues similar to the ones found in [IERC20.approve()](https://docs.openzeppelin.com/contracts/4.x/api/token/erc20#IERC20-approve-address-uint256-), and its usage is discouraged.

Whenever possible, use [`safeIncreaseAllowance()`](https://docs.openzeppelin.com/contracts/4.x/api/token/erc20#SafeERC20-safeIncreaseAllowance-contract-IERC20-address-uint256-) and [`safeDecreaseAllowance()`](https://docs.openzeppelin.com/contracts/4.x/api/token/erc20#SafeERC20-safeDecreaseAllowance-contract-IERC20-address-uint256-) instead.

Additionally, some tokens, e.g USDT, do not work when changing the allowance from an existing non-zero allowance value. For instance, Tether (USDT)'s approve() will revert if the current approval has not been set to zero, serving to protect against front-running changes of approvals.

## Vulnerability Detail
In the protocol, all functions using `safeapprove()` must be first approved by zero on top getting it replaced by `safeDecreaseAllowance()`. These are `ensureApprove()` in BasicSpell.sol, `initialize()` in SoftVault.sol, `mint()` and `burn()` in WichiFarm.sol. 

## Impact
Otherwise, these functions are going to revert every time they encounter such kind of tokens that might have a remaining allowance (even in dust amount) associated.

## Code Snippet
[File: BasicSpell.sol#L47-L52](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/BasicSpell.sol#L47-L52)

```solidity
    function ensureApprove(address token, address spender) internal {
        if (!approved[token][spender]) {
            IERC20Upgradeable(token).safeApprove(spender, type(uint256).max);
            approved[token][spender] = true;
        }
    }
```
[File: SoftVault.sol#L41-L56](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/vault/SoftVault.sol#L41-L56)

```solidity
    function initialize(
        IProtocolConfig _config,
        ICErc20 _cToken,
        string memory _name,
        string memory _symbol
    ) external initializer {
        __ERC20_init(_name, _symbol);
        __Ownable_init();
        if (address(_cToken) == address(0) || address(_config) == address(0))
            revert ZERO_ADDRESS();
        IERC20Upgradeable _uToken = IERC20Upgradeable(_cToken.underlying());
        config = _config;
        cToken = _cToken;
        uToken = _uToken;
        _uToken.safeApprove(address(_cToken), type(uint256).max);
    }
```
[File: WIchiFarm.sol#L82-L135](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/wrapper/WIchiFarm.sol#L82-L135)

```solidity
    function mint(uint256 pid, uint256 amount)
        external
        nonReentrant
        returns (uint256)
    {
        address lpToken = ichiFarm.lpToken(pid);
        IERC20Upgradeable(lpToken).safeTransferFrom(
            msg.sender,
            address(this),
            amount
        );
        if (
            IERC20Upgradeable(lpToken).allowance(
                address(this),
                address(ichiFarm)
            ) != type(uint256).max
        ) {
            // We only need to do this once per pool, as LP token's allowance won't decrease if it's -1.
            IERC20Upgradeable(lpToken).safeApprove(
                address(ichiFarm),
                type(uint256).max
            );
        }
        ichiFarm.deposit(pid, amount, address(this));
        (uint256 ichiPerShare, , ) = ichiFarm.poolInfo(pid);
        uint256 id = encodeId(pid, ichiPerShare);
        _mint(msg.sender, id, amount, "");
        return id;
    }

    function burn(uint256 id, uint256 amount)
        external
        nonReentrant
        returns (uint256)
    {
        if (amount == type(uint256).max) {
            amount = balanceOf(msg.sender, id);
        }
        (uint256 pid, uint256 stIchiPerShare) = decodeId(id);
        _burn(msg.sender, id, amount);

        uint256 ichiRewards = ichiFarm.pendingIchi(pid, address(this));
        ichiFarm.harvest(pid, address(this));
        ichiFarm.withdraw(pid, amount, address(this));

        // Convert Legacy ICHI to ICHI v2
        if (ichiRewards > 0) {
            ICHIv1.safeApprove(address(ICHI), ichiRewards);
            ICHI.convertToV2(ichiRewards);
        }
```
## Tool used

Manual Review

## Recommendation
Consider approving 0 first prior to using the recommended `safeIncreaseAllowance()` to set the value of allowances.

For example, the first instance in the Code Snippet may be refactored as follows:

```diff
    function ensureApprove(address token, address spender) internal {
        if (!approved[token][spender]) {
+            IERC20Upgradeable(token).safeApprove(spender, 0);
+            IERC20Upgradeable(token).safeIncreaseAllowance(spender, type(uint256).max);
-            IERC20Upgradeable(token).safeApprove(spender, type(uint256).max);
            approved[token][spender] = true;
        }
    }
```