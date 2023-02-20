berndartmueller

medium

# Burning `WIchiFarm` wrapped ICHI vault LP tokens can possibly revert due to insufficient balance of ICHI v2 tokens

> Note: This issue is only applicable if the `stIchi` and `enIchi` values have their precision increased to **18 decimals**. In the current implementation, those values are incorrectly in **9 decimals**! The issue has been reported as a separate issue. Nevertheless, I consider this a valid issue, and due to the dependence on the other issue, I'm reporting it as Medium severity.

## Summary

Calculating the eligible user rewards with a higher precision of 18 decimals can cause the token transfer to revert as fewer farming rewards have been received before (due to the `ICHI` v1 token using 9 decimals and the `ICHI` v2 token using 18 decimals).

## Vulnerability Detail

Closing an ICHI vault spell farming position burns the wrapped ICHI vault LP tokens (`WIchiFarm` ERC-1155 tokens). Farming rewards are harvested from the ICHI farm ([see contract on Etherscan](https://etherscan.io/address/0x275dfe03bc036257cd0a713ee819dbd4529739c8)) and received as `ICHI` **v1** tokens.

The `ICHI` **v1** ERC-20 token uses **9 decimals** ([see token on Etherscan](https://etherscan.io/token/0x903bEF1736CDdf2A537176cf3C64579C3867A881)), whereas the `ICHI` **v2** ERC-20 token uses **18 decimals** ([see token on Etherscan](https://etherscan.io/token/0x111111517e4929D3dcbdfa7CCe55d30d4B6BC4d6)).

Those received `ICHI` **v1** tokens are then converted to **v2** tokens in line 134. The conversion simply multiplies the `ICHI` **v1** token amount by `1e9` (9 decimals) to scale the value to 18 decimals.

The user's share of eligible `ICHI` v2 reward tokens is calculated as the delta of `enIchi` and `stIchi`. However, those two values are in **18 decimal** precision (once corrected, as noted in a separate issue). As the harvested `ICHI` **v1** tokens are scaled to 18 decimals, there is a slight precision loss and truncation.

**For example, consider the very simplified foundry example:**

```solidity
uint256 stIchiPerShare = 0.01e9;
uint256 amount = 1.00000001e18;
uint256 rewardV1 = (stIchiPerShare * amount) / 1e18;
uint256 rewardV2 = (stIchiPerShare * amount) / 1e9;

console.log("rewardV1:", rewardV1);
console.log("rewardV1 converted to V2:", rewardV1 * 1e9);
console.log("rewardV2:", rewardV2);

assertEq(rewardV2, rewardV1 * 1e9);
```

The result:

```bash
rewardV1: 10000000
rewardV1 converted to V2: 10000000000000000
rewardV2: 10000000100000000
Error: a == b not satisfied [uint]
  Expected: 10000000000000000
    Actual: 10000000100000000
```

The `amount` uses a precision of 18 decimals and therefore the `rewardV2` value is more precise than the `rewardV1` value.

The `ICHI` v2 token transfer in line 147 can therefore potentially revert due to insufficient balance, caused by the precision and truncation issue.

## Impact

Unwrapping `WIchiFarm` tokens will revert due to transferring more ICHI v2 tokens than available. The last user to burn those tokens (while closing a Ichi vault spell position) will be unable to do so and will be unable to close the position. The isolated collateral will be locked.

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
144:     uint256 enIchi = (enIchiPerShare * amount) / 1e18;
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

Consider checking the current balance of `ICHI` v2 tokens before trying to transfer more than available and capping the transfer amount to the current balance.
