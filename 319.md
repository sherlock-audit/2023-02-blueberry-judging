berndartmueller

high

# Too few `ICHI` v2 farming reward tokens transferred to the user due to incorrect decimal precision

## Summary

The `burn` function in the `WIchiFarm` contract transfers too few `ICHI` **v2** farming reward tokens to the caller due to using 9 decimals instead of 18 decimals for the `ICHI` **v2** token.

## Vulnerability Detail

Closing an ICHI vault spell farming position burns the wrapped ICHI vault LP tokens (`WIchiFarm` ERC-1155 tokens). Farming rewards are harvested from the ICHI farm ([see contract on Etherscan](https://etherscan.io/address/0x275dfe03bc036257cd0a713ee819dbd4529739c8)) and received as `ICHI` **v1** tokens.

The `ICHI` **v1** ERC-20 token uses **9 decimals** ([see token on Etherscan](https://etherscan.io/token/0x903bEF1736CDdf2A537176cf3C64579C3867A881)), whereas the `ICHI` **v2** ERC-20 token uses **18 decimals** ([see token on Etherscan](https://etherscan.io/token/0x111111517e4929D3dcbdfa7CCe55d30d4B6BC4d6)).

Those received `ICHI` **v1** tokens are then converted to **v2** tokens in line 134.

To calculate the user's share of eligible `ICHI` **v2** reward tokens, the reward per share accumulator `stIchiPerShare` at the time of minting the `WIchiFarm` token and the current `enIchiPerShare` accumulator is used.

However, those accumulator values are in **9 decimals** precision (please see the `ichiFarmV2.harvest` function for proof that `pool.accIchiPerShare` uses 9 decimals, otherwise the `ICHI` token transfer would fail due to inflated `_pendingIchi`). Given that `amount` is in **18 decimals**, the calculation of `stIchi` and `enIchi` in lines 143 and 144 will result in a value with **9 decimals** precision.

As previously mentioned, the `ICHI` **v2** token uses **18 decimals**. Therefore, too few `ICHI` **v2** tokens are transferred.

## Impact

Users will receive substantially fewer `ICHI` v2 farming reward tokens than expected.

## Code Snippet

[wrapper/WIchiFarm.sol#L143-L144](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/wrapper/WIchiFarm.sol#L143-L144)

```solidity
116: function burn(uint256 id, uint256 amount)
117:     external
118:     nonReentrant
119:     returns (uint256)
120: {
121:     if (amount == type(uint256).max) {
122:         amount = balanceOf(msg.sender, id);
123:     }
124:     (uint256 pid, uint256 stIchiPerShare) = decodeId(id);
125:     _burn(msg.sender, id, amount);
126:
127:     uint256 ichiRewards = ichiFarm.pendingIchi(pid, address(this));
128:     ichiFarm.harvest(pid, address(this));
129:     ichiFarm.withdraw(pid, amount, address(this));
130:
131:     // Convert Legacy ICHI to ICHI v2
132:     if (ichiRewards > 0) {
133:         ICHIv1.safeApprove(address(ICHI), ichiRewards);
134:         ICHI.convertToV2(ichiRewards);
135:     }
136:
137:     // Transfer LP Tokens
138:     address lpToken = ichiFarm.lpToken(pid);
139:     IERC20Upgradeable(lpToken).safeTransfer(msg.sender, amount);
140:
141:     // Transfer Reward Tokens
142:     (uint256 enIchiPerShare, , ) = ichiFarm.poolInfo(pid);
143:     uint256 stIchi = (stIchiPerShare * amount).divCeil(1e18);
144:     uint256 enIchi = (enIchiPerShare * amount) / 1e18; // @audit-info `enIchi` and `stIchi` are in 9 decimal precision
145:
146:     if (enIchi > stIchi) {
147:         ICHI.safeTransfer(msg.sender, enIchi - stIchi);
148:     }
149:     return pid;
150: }
```

## Tool used

Manual Review

## Recommendation

Consider changing the denominator in lines 143 and 144 from `1e18` to `1e9` to use the required `18` decimals for the `ICHI` v2 token.
