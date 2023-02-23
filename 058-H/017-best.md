0x52

high

# WIchiFarm#burn sends too few IchiV2 tokens to users

## Summary

IchiV1 is a 9dp token and IchiV2 is an 18dp token. IchiFarmV2 distributes and tracks rewards in IchiV1. The issue is that WIchiFarm converts the V1 tokens received to V2 tokens but then uses the 9dp accIchiPerShare to determine the number of tokens to send to the user. The result is that the end user receives a fraction of the tokens they should and the other tokens are trapped forever in the contract.

## Vulnerability Detail

        uint256 ichiRewards = ichiFarm.pendingIchi(pid, address(this));
        ichiFarm.harvest(pid, address(this));
        ichiFarm.withdraw(pid, amount, address(this));

        // Convert Legacy ICHI to ICHI v2
        if (ichiRewards > 0) {
            ICHIv1.safeApprove(address(ICHI), ichiRewards);
            ICHI.convertToV2(ichiRewards);
        }

        // Transfer LP Tokens
        address lpToken = ichiFarm.lpToken(pid);
        IERC20Upgradeable(lpToken).safeTransfer(msg.sender, amount);

        // Transfer Reward Tokens
        (uint256 enIchiPerShare, , ) = ichiFarm.poolInfo(pid);
        uint256 stIchi = (stIchiPerShare * amount).divCeil(1e18);
        uint256 enIchi = (enIchiPerShare * amount) / 1e18;

        if (enIchi > stIchi) {
            ICHI.safeTransfer(msg.sender, enIchi - stIchi);
        }

Rewards collected from IchiFarm are paid as IchiV1 tokens, which has 9 dp. These rewards are converted to IchiV2 tokens before being sent to the user. The issue is that IchiV2 is an 18 dp token. When sending the tokens it sends the 9 dp IchiV1 token amount to the user. This means that only a fraction of the tokens will be sent to the user and all the other tokens will be irretrievable.

    function harvest(uint256 pid, address to) external {
        require(!nonReentrant, "ichiFarmV2::nonReentrant - try again");
        nonReentrant = true;

        PoolInfo memory pool = updatePool(pid);
        UserInfo storage user = userInfo[pid][msg.sender];
        int256 accumulatedIchi = int256(user.amount.mul(pool.accIchiPerShare) / ACC_ICHI_PRECISION);
        uint256 _pendingIchi = accumulatedIchi.sub(user.rewardDebt).toUInt256();

        // Effects
        user.rewardDebt = accumulatedIchi;

        // Interactions
        if (_pendingIchi > 0) {
            ICHI.safeTransfer(to, _pendingIchi);
        }

        emit Harvest(msg.sender, pid, _pendingIchi);
        nonReentrant = false;
    }

Above is the harvest function for the IchiFarm, which will be used for the example below.

Example:
Imagine a user deposits 1e18 LP tokens when accIchiPerShare = 0. This gives stIchiPerShare = 0 and rewardDebt = 0. Now the accIchiPerShare increases to 1e9 and the user withdraws. This triggers the harvest function which transfers IchiV1 to WIchiFarm:

        int256 accumulatedIchi = 1e18 * 1e9 / 1e18 = 1e9
        uint256 _pendingIchi = 1e9 - 0 = 1e9

This will transfer 1e9 IchiV1 to WIchiFarm. This will be redeemed for IchiV2 which has 18 dp:

        1e9 IchiV1 => 1e18 IchiV2

Next we calculate the amount of IchiV2 send to the user:

        uint256 stIchi = 0 * 1e18 / 1e18 = 0
        uint256 enIchi = 1e9 * 1e18 / 1e18 = 1e9
        
        transferAmount = stIchi - enIchi = 1e9

We can see that the contract will send the user 1e9 IchiV2 when it should be sending 1e18.

The extra IchiV2 that collects in the contract is irretrievable leading to near entire loss of Ichi rewards for all users who use WIchiFarm for their LP.

## Impact

Loss of nearly all Ichi token rewards for LP holders

## Code Snippet

https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/wrapper/WIchiFarm.sol#L116-L150

## Tool used

Manual Review

## Recommendation

The reward token amount needs to be scaled by 1e9 to make sure the user gets the proper amount of rewards:

        // Transfer Reward Tokens
        (uint256 enIchiPerShare, , ) = ichiFarm.poolInfo(pid);
        uint256 stIchi = (stIchiPerShare * amount).divCeil(1e18);
        uint256 enIchi = (enIchiPerShare * amount) / 1e18;

        if (enIchi > stIchi) {
    -       ICHI.safeTransfer(msg.sender, enIchi - stIchi);
    +       ICHI.safeTransfer(msg.sender, (enIchi - stIchi) * 10**9);
        }