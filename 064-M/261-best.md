HonorLt

false

# Low severity issues

## Summary

I have grouped some low-severity issues that do not pretend to be rewarded but might still be useful and considerable fixing.

## Vulnerability Detail

1. Missing `__ReentrancyGuard_init`:
```solidity
    function initialize(
        IProtocolConfig _config,
        ICErc20 _cToken,
        string memory _name,
        string memory _symbol
    ) external initializer {
        __ERC20_init(_name, _symbol);
        __Ownable_init();
```

2. I believe the condition should not be strict ```>=``` here because if my position is right at the threshold I should not be liquidatable:
```solidity
    function isLiquidatable(uint256 positionId)
        public
        view
        returns (bool liquidatable)
    {
        Position storage pos = positions[positionId];
        uint256 risk = getPositionRisk(positionId);
        liquidatable = risk >= oracle.getLiqThreshold(pos.underlyingToken);
    }
```

3. Unsafe casting to `int256(amountToSwap)`. Not likely to be exploited in practice but technically can happen with tokens that have very large supplies or e.g. rebasing mechanism:
```solidity
uint256 amountToSwap = IERC20(
            isTokenA ? vault.token1() : vault.token0()
        ).balanceOf(address(this));
        if (amountToSwap > 0) {
            swapPool = IUniswapV3Pool(vault.pool());
            swapPool.swap(
                address(this),
                // if withdraw token is Token0, then swap token1 -> token0 (false)
                !isTokenA,
                int256(amountToSwap),
                isTokenA
                    ? UniV3WrappedLibMockup.MAX_SQRT_RATIO - 1 // Token0 -> Token1
                    : UniV3WrappedLibMockup.MIN_SQRT_RATIO + 1, // Token1 -> Token0
                abi.encode(address(this))
            );
        }
```

4. `safeApprove` will revert if the current approval is not 0, must reset to 0 first. Again, not likely to happen in practice because `ichiFarm.deposit` does not reduce the approval:
```solidity
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
```

5.  You might consider adding pagination to support efficient querying of large data sets:
```solidity
    function getPositionIdsByOwner(address owner)
        external
        view
        returns (uint256[] memory ids)
    {
        uint256[] memory matchingIds = new uint256[](nextPositionId);
        uint256 index;
        for (uint256 i = 0; i < nextPositionId; i++) {
            if (positions[i].owner == owner) {
                matchingIds[index] = i;
                index++;
            }
        }

        ids = new uint256[](index);
        for (uint256 i = 0; i < index; i++) {
            ids[i] = matchingIds[i];
        }
    }
```

6. `_setPrimarySources` could validate that the same source is not added twice:
```solidity
    function _setPrimarySources(
        address token,
        uint256 maxPriceDeviation,
        IBaseOracle[] memory sources
    ) internal {
       ...
        for (uint256 idx = 0; idx < sources.length; idx++) {
            if (address(sources[idx]) == address(0)) revert ZERO_ADDRESS();
            primarySources[token][idx] = sources[idx];
        }
        emit SetPrimarySources(token, maxPriceDeviation, sources);
    }
```

## Impact

No real serious harm to the system. Would require certain conditions to trigger these corner cases.

## Code Snippet

1.
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/vault/HardVault.sol#L36-L38
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/vault/SoftVault.sol#L41-L48
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/wrapper/WERC20.sol#L15-L17
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/wrapper/WIchiFarm.sol#L30-L39

2.
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L504

3.
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L311

4.
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/wrapper/WIchiFarm.sol#L93-L104

5.
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L315-L333

6.
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/AggregatorOracle.sol#L79-L82

## Tool used

Manual Review

## Recommendation

Consider fixing or ignoring the aforementioned issues.
