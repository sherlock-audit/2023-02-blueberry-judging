0Kage

high

# Users lose part of `ICHI` rewards when they attempt to 'take' partial collateral by calling `closePositionFarm`

## Summary
Users can 'take' collateral by passing `lpTakeAmt` in `closePositionFarm`. `lpTakeAmt` in this case refers to wrapped farm tokens minted by `wIchiFarm` address. After taking back collateral from the bank, `wIchiFarm` calls the `burn()` function to harvest rewards and transfer them back to position owner.

`harvest` function of `ichiFarm` transfers the entire pending ITCHI rewards to the sender (which in this case is `wIchiFarm` address), regardless of the burn amount requested.

Of this, only proportionate amount of rewards are transfered back to the `ichiVaultSpell` - proportion here is determined by the `lpTakeAmt`. Balance tokens that rightfully belong to the position owner remain forvever trapped inside the `ichiVaultSpell` contract.

## Vulnerability Detail
On sending a request to take partial collateral, `closePositionFarm` function calls the [`burn` function on `wIchiFarm`](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/wrapper/WIchiFarm.sol#L128). `burn()` calls the `harvest` function to collect all rewards from `wIchiFarm`.

On checking codebase of [ichiFarm V2](https://github.com/ichifarm/ichi-farming/blob/main/contracts/ichiFarmV2.sol#L252), we notice that `harvest` function transfers all rewards back to sender (which is `wIchiFarm`). Note that, even though we are burning partial tokens, `harvest` function returns rewards on the entire LP tokens present in farming contract.

```solidity
function harvest(uint256 pid, address to) external {
    ...

      uint256 _pendingIchi = accumulatedIchi.sub(user.rewardDebt).toUInt256();

        // Effects
        user.rewardDebt = accumulatedIchi;

        // Interactions
        if (_pendingIchi > 0) {
            ICHI.safeTransfer(to, _pendingIchi);
        }

    ...

}
```
Now, coming back to the [`burn()` function in `wIchiFarm`](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/wrapper/WIchiFarm.sol#L142), notice that rewards only proportionate to burn amount are being transfered back to the `ichiVaultSpell` that will further transfer them to position owner.

```solidity

 function burn(uint256 id, uint256 amount)
        external
        nonReentrant
        returns (uint256)
    {

        ...

        ichiFarm.harvest(pid, address(this));

        ...

         // Transfer Reward Tokens
        (uint256 enIchiPerShare, , ) = ichiFarm.poolInfo(pid);
        uint256 stIchi = (stIchiPerShare * amount).divCeil(1e18);
        uint256 enIchi = (enIchiPerShare * amount) / 1e18;

        if (enIchi > stIchi) {
            ICHI.safeTransfer(msg.sender, enIchi - stIchi);
        }
        ...
    }

```
So the balance rewards are stuck inside the spell contract & there is no way for users to get back the rewards on subsequent burning of farm tokens.


## Impact
Users taking partial collateral will lose part of their rewards permanently.

POC

- Alice is an existing borrower with following positions (1 ETH - underlying, 700 USDC borrowing, collateral: 400 ICHI farm tokens)

- Alice requests to take back 100 ICHI farm tokens

- 100 ICHI farm tokens are burnt - and say all 400 ICHI farm tokens have collected 30 ICHI reward tokens

- `wIchiFarm` wrapper gets back all 30 ICHI rewards

- Wrapper sends back only 7.5 ICHI rewards (25% of total rewards corresponding to 25% of ICHI farm tokens burnt) to `ichiVaultSpell`

- `ichiVaultSpell` sends back 7.5 ICHI rewards to Alice

- Alice loses balance 22.5 ICHI rewards which are permanently trapped inside the `wIchiFarm` wrapper

## Code Snippet
https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/wrapper/WIchiFarm.sol#L128

## Tool used
Manual Review

## Recommendation
Transfer back all rewards to `ichiVaultSpell` which inturn transfers them back to the owner.

Inside the [`burn()` function](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/wrapper/WIchiFarm.sol#L142), replace code block where rewards propotionate to `amount` are calculated....

```solidity
        (uint256 enIchiPerShare, , ) = ichiFarm.poolInfo(pid);
        uint256 stIchi = (stIchiPerShare * amount).divCeil(1e18);
        uint256 enIchi = (enIchiPerShare * amount) / 1e18;

        if (enIchi > stIchi) {
            ICHI.safeTransfer(msg.sender, enIchi - stIchi);
        }
```

... with `ichiRewards` instead - this way all rewards upto that point are sent back to owner

```solidity
            ICHI.safeTransfer(msg.sender, ichiRewards); //**** - audit - ichiRewards are the entire pending rewards upto this point****
```
