seeu

medium

# Contracts have a single point of control

## Summary

Contracts have a single point of control

## Vulnerability Detail

When contracts have a single point of control, contract owners need to be trusted to prevent fraudulent upgrades and money draining since they have privileged access to carry out administrative chores.

## Impact

Owners could use their privileges to perform malicious actions like drain funds.

More care needs to be taken in contracts like [ProtocolConfig.sol](https://github.com/sherlock-audit/2023-02-blueberry-seeu-inspace/tree/main/contracts/ProtocolConfig.sol) in functions like `setTreasuryWallet`. In this example, it is possible for the owner to set the wallet address for treasury.

## Code Snippet


- [contracts/BlueBerryBank.sol](https://github.com/sherlock-audit/2023-02-blueberry-seeu-inspace/tree/main/contracts/BlueBerryBank.sol)
  ```Solidity
  #L126 =>     function setAllowContractCalls(bool ok) external onlyOwner {
  #L136 =>     ) external onlyOwner {
  #L154 =>     ) external onlyOwner {
  #L172 =>     ) external onlyOwner {
  #L221 =>     function setBankStatus(uint256 _bankStatus) external onlyOwner {
  ```
- [contracts/ProtocolConfig.sol](https://github.com/sherlock-audit/2023-02-blueberry-seeu-inspace/tree/main/contracts/ProtocolConfig.sol)
  ```Solidity
  #L43 =>     function startVaultWithdrawFee() external onlyOwner {
  #L50 =>     function setDepositFee(uint256 depositFee_) external onlyOwner {
  #L56 =>     function setWithdrawFee(uint256 withdrawFee_) external onlyOwner {
  #L66 =>     ) external onlyOwner {
  #L76 =>     function setTreasuryWallet(address treasury_) external onlyOwner {
  #L81 =>     function setBlbUsdcIchiVault(address vault_) external onlyOwner {
  #L86 =>     function setBlbStabilityPool(address pool_) external onlyOwner {
  ```
- [contracts/mock/MockFeedRegistry.sol](https://github.com/sherlock-audit/2023-02-blueberry-seeu-inspace/tree/main/contracts/mock/MockFeedRegistry.sol#L54)
  ```Solidity
  #L54 =>     ) external onlyOwner {
  ```
- [contracts/mock/MockIchiFarm.sol](https://github.com/sherlock-audit/2023-02-blueberry-seeu-inspace/tree/main/contracts/mock/MockIchiFarm.sol)
  ```Solidity
  #L98 =>         onlyOwner
  #L109 =>     function setNonReentrant(bool _val) external onlyOwner returns (bool) {
  #L135 =>     function add(uint256 allocPoint, IERC20 _lpToken) external onlyOwner {
  #L158 =>     function set(uint256 _pid, uint256 _allocPoint) external onlyOwner {
  ```
- [contracts/mock/MockIchiVault.sol](https://github.com/sherlock-audit/2023-02-blueberry-seeu-inspace/tree/main/contracts/mock/MockIchiVault.sol)
  ```Solidity
  #L178 =>     function setTwapPeriod(uint32 newTwapPeriod) external onlyOwner {
  #L351 =>     ) external override nonReentrant onlyOwner {
  #L446 =>     function setFactory(address _newFactory) external onlyOwner {
  #L637 =>     function setMaxTotalSupply(uint256 _maxTotalSupply) external onlyOwner {
  #L647 =>     function setHysteresis(uint256 _hysteresis) external onlyOwner {
  #L657 =>     function setAffiliate(address _affiliate) external override onlyOwner {
  #L671 =>         onlyOwner
  ```
- [contracts/mock/MockOracle.sol](https://github.com/sherlock-audit/2023-02-blueberry-seeu-inspace/tree/main/contracts/mock/MockOracle.sol#L27)
  ```Solidity
  #L27 =>         onlyOwner
  ```
- [contracts/oracle/AggregatorOracle.sol](https://github.com/sherlock-audit/2023-02-blueberry-seeu-inspace/tree/main/contracts/oracle/AggregatorOracle.sol)
  ```Solidity
  #L35 =>     ) external onlyOwner {
  #L47 =>     ) external onlyOwner {
  ```
- [contracts/oracle/BandAdapterOracle.sol](https://github.com/sherlock-audit/2023-02-blueberry-seeu-inspace/tree/main/contracts/oracle/BandAdapterOracle.sol)
  ```Solidity
  #L27 =>     function setRef(IStdReference _ref) external onlyOwner {
  #L38 =>         onlyOwner
  ```
- [contracts/oracle/BaseAdapter.sol](https://github.com/sherlock-audit/2023-02-blueberry-seeu-inspace/tree/main/contracts/oracle/BaseAdapter.sol#L20)
  ```Solidity
  #L20 =>     ) external onlyOwner {
  ```
- [contracts/oracle/ChainlinkAdapterOracle.sol](https://github.com/sherlock-audit/2023-02-blueberry-seeu-inspace/tree/main/contracts/oracle/ChainlinkAdapterOracle.sol)
  ```Solidity
  #L37 =>     function setFeedRegistry(IFeedRegistry _registry) external onlyOwner {
  #L50 =>     ) external onlyOwner {
  ```
- [contracts/oracle/CoreOracle.sol](https://github.com/sherlock-audit/2023-02-blueberry-seeu-inspace/tree/main/contracts/oracle/CoreOracle.sol)
  ```Solidity
  #L40 =>         onlyOwner
  #L58 =>     ) external onlyOwner {
  #L74 =>     function removeTokenSettings(address[] memory tokens) external onlyOwner {
  #L86 =>         onlyOwner
  ```
- [contracts/oracle/UniswapV3AdapterOracle.sol](https://github.com/sherlock-audit/2023-02-blueberry-seeu-inspace/tree/main/contracts/oracle/UniswapV3AdapterOracle.sol#L28)
  ```Solidity
  #L28 =>         onlyOwner
  ```
- [contracts/spell/IchiVaultSpell.sol](https://github.com/sherlock-audit/2023-02-blueberry-seeu-inspace/tree/main/contracts/spell/IchiVaultSpell.sol)
  ```Solidity
  #L77 =>     function addStrategy(address vault, uint256 maxPosSize) external onlyOwner {
  #L88 =>     ) external existingStrategy(strategyId) onlyOwner {
  ```

## Tool used

Manual Review

## Recommendation

Some solutions include:
- implementing timelocks
- multi signature custody

Reference:
- [What is Centralization Risk?](https://certik.medium.com/what-is-centralization-risk-41cf848f5a74)
