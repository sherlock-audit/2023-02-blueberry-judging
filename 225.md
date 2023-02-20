RaymondFam

medium

# Front-running of initialization

## Summary
The protocol's initialization functions are generally unprotected with no access controls in place, and can be front-run by anyone.

## Vulnerability Detail
The contract instances associated with the affected `initialize()` are:

- BlueBerryBank.sol
- ProtocolConfig.sol
- CoreOracle.sol
- IchiVaultSpell.sol
- HardVault.sol
- SoftVault.sol
- WERC20.sol
- WIchiFarm.sol

## Impact
The init parameters can be set only once in the functions above because of initializer and will be permanently determined by the attacker unless there are setter functions catered for the parameters involved. The situations could be worse if the initializations were to entail setting default admin and critical roles to the contract.

## Code Snippet
[File: BlueBerryBank.sol#L94-L113](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L94-L113)

```solidity
    function initialize(IOracle _oracle, IProtocolConfig _config)
        external
        initializer
    {
        __Ownable_init();
        if (address(_oracle) == address(0) || address(_config) == address(0)) {
            revert ZERO_ADDRESS();
        }
        _GENERAL_LOCK = _NOT_ENTERED;
        _IN_EXEC_LOCK = _NOT_ENTERED;
        POSITION_ID = _NO_ID;
        SPELL = _NO_ADDRESS;

        config = _config;
        oracle = _oracle;
        nextPositionId = 1;
        bankStatus = 7; // allow borrow, lend, repay

        emit SetOracle(address(_oracle));
    }
```
[File: ProtocolConfig.sol#L28-L41](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/ProtocolConfig.sol#L28-L41)

```solidity
    function initialize(address treasury_) external initializer {
        __Ownable_init();
        if (treasury_ == address(0)) revert ZERO_ADDRESS();
        treasury = treasury_;

        depositFee = 50; // 0.5% as default, base 10000
        withdrawFee = 50; // 0.5% as default, base 10000
        treasuryFeeRate = 3000; // 30% of deposit/withdraw fee => 0.15%
        blbStablePoolFeeRate = 3500; //  35% of deposit/withdraw fee => 0.175%
        blbIchiVaultFeeRate = 3500; //  35% of deposit/withdraw fee => 0.175%

        withdrawVaultFee = 100; // 1% as default, base 10000
        withdrawVaultFeeWindow = 60 days;
    }
```
[File: CoreOracle.sol#L31-L33](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/CoreOracle.sol#L31-L33)

```solidity
    function initialize() external initializer {
        __Ownable_init();
    }
```
[File: IchiVaultSpell.sol#L59-L70](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L59-L70)

```solidity
    function initialize(
        IBank _bank,
        address _werc20,
        address _weth,
        address _wichiFarm
    ) external initializer {
        __BasicSpell_init(_bank, _werc20, _weth);

        wIchiFarm = IWIchiFarm(_wichiFarm);
        ICHI = address(wIchiFarm.ICHI());
        IWIchiFarm(_wichiFarm).setApprovalForAll(address(_bank), true);
    }
```
[File: HardVault.sol#L36-L41](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/vault/HardVault.sol#L36-L41)

```solidity
    function initialize(IProtocolConfig _config) external initializer {
        __ERC1155_init("HardVault");
        __Ownable_init();
        if (address(_config) == address(0)) revert ZERO_ADDRESS();
        config = _config;
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
[File: WERC20.sol#L15-L17](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/wrapper/WERC20.sol#L15-L17)

```solidity
    function initialize() external initializer {
        __ERC1155_init("WERC20");
    }
```
[File: WIchiFarm.sol#L30-L39](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/wrapper/WIchiFarm.sol#L30-L39)

```solidity
    function initialize(
        address _ichi,
        address _ichiv1,
        address _ichiFarm
    ) external initializer {
        __ERC1155_init("WIchiFarm");
        ICHI = IIchiV2(_ichi);
        ICHIv1 = IERC20Upgradeable(_ichiv1);
        ichiFarm = IIchiFarm(_ichiFarm);
    }
```
## Tool used

Manual Review

## Recommendation
As per Openzeppelin's recommendation:

https://forum.openzeppelin.com/t/uupsupgradeable-vulnerability-post-mortem/15680/6

The guidelines are now to prevent front-running of initialize() on an implementation contract, by adding an empty constructor with the initializer modifier. Hence, the implementation contract gets initialized atomically upon deployment.

This feature is readily incorporated in the Solidity Wizard since the UUPS vulnerability discovery. You would just need to check UPGRADEABILITY to have a specific constructor code block added to the contract.

```solidity
    /// @custom:oz-upgrades-unsafe-allow constructor
    constructor() {
        _disableInitializers();
    }
```
Alternatively, consider:
● adding access controls to initialize(), or
● ensuring that the documentation clearly warns users about incorrect initialization.

Long term, avoid initialization outside of the constructor. If that is not possible, ensure that the underlying risks of initialization are documented and properly tested.