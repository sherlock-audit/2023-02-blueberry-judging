tsvetanovv

medium

# Malicious user can Blocklists Token

## Summary
The protocol currently uses these tokens:

> ERC20: [whitelisted - current list of supported assets: USDC, DAI, ALCX, BAL, CRV, ICHI, SUSHI, WBTC, WETH]

Some tokens (e.g. USDC, USDT) have a contract level admin controlled address blocklist. If an address is blocked, then transfers to and from that address are forbidden.

## Vulnerability Detail
There are currently 200+ blacklisted accounts by USDC, these accounts are related to known hacks and other crime events.
https://etherscan.io/address/0x5db0115f3b72d19cea34dd697cf412ff86dc7e1b.

## Impact
Malicious or compromised token owners can trap funds in a contract by adding the contract address to the blocklist.

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L22
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/BasicSpell.sol
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol

## Tool used

Manual Review

## Recommendation

Try to implement a try-catch solution where you skip certain funds whenever they cause the USDC transfer to revert.