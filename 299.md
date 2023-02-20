Jeiwan

medium

# Spent token approvals can potentially cause indefinite DoS

## Summary
The amounts approved by the `ensureApprove` function can be spent, leading to the blocking of token transfers in spells due do the function being called only once per a token-spender pair.
## Vulnerability Detail
The [ensureApprove](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/BasicSpell.sol#L47) function approve spending of a token to a spender: it approves spending of 2**256 - 1 tokens once per token and spender. However, some tokens the function is used with (namely, [USDC](https://etherscan.io/address/0xa2327a938febf5fec13bacfb16ae10ecbc4cbdcf#code), [ALCX](https://etherscan.io/token/0xdbdb4d16eda451d0503b854cf79d55697f90c8df#code), [BAL](https://etherscan.io/token/0xba100000625a3754423978a60c9317c58a424e3d#code), [CRV](https://etherscan.io/token/0xD533a949740bb3306d119CC777fa900bA034cd52#code), [SUSHI](https://etherscan.io/token/0x6b3595068778dd592e39a122f4f5a5cf09c90fe2#code), [WBTC](https://etherscan.io/token/0x2260fac5e5542a773aa44fbcfedf7c193bc2c599#code), and [ICHI_Vault_LP](https://etherscan.io/address/0x683F081DBC729dbD34AbaC708Fa0B390d49F1c39#code)) reduce the approved amount in calls to `transferFrom`. Approved amounts of these tokens will reduce over time: the longer the contracts are active and the higher the activity of Blueberry users, the faster the approved amounts will be spent.
## Impact
Token transfers in spells may be blocked, leading to indefinite blocking of [loan repayments](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/BasicSpell.sol#L110), [addition of collateral](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/BasicSpell.sol#L120), [borrowing](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L141), and [deposits to farming pools](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/IchiVaultSpell.sol#L245).
## Code Snippet
[BasicSpell.sol#L48-L51](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/spell/BasicSpell.sol#L48-L51)
## Tool used
Manual Review
## Recommendation
Instead of approving spending of tokens only once, consider checking the current approved amount and updating the approved amount if's not enough for a transfer.