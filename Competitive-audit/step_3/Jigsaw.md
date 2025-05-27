**Title: Jigsaw**

Auditor: **vesko210**

## Issues found
|Severtity|Number of issues found
| ------- | -------------------- |
| High    | 1                    |
| Medium  | 0                    |
| Low     | 0                    |

# Findings

## [H-1] Rewards will get stuck into the contract when there are no stakers

## Summary

Rewards are not being accumulated for the time when there are no stakers, which will leed to funds being locked and inaccessible.

---

## Finding Description

The `Staker` contract calculates `rewardRate` using integer division, which introduces rounding errors when `_amount + leftover` is not perfectly divisible by `rewardsDuration`. Additionally, and most importantly, the contract does not accumulate rewards when there are no stakers. This leads to a period in the where no rewards are being accumulated. As a result, the leftover tokens are not being redistributed to the stakers nor claimed back to the owner.

The issue will ocurre every time when the period has finished and new rewards are added. Because it does not add the `balanceOf(address(this))`  to the `_amount` to be distributed.

```solidity
    if (block.timestamp >= periodFinish) {
        // It does not add the not allocated rewards to the new rewards amount
        rewardRate = _amount / rewardsDuration;
    } else {
        uint256 remaining = periodFinish - block.timestamp;
        uint256 leftover = remaining * rewardRate;
        rewardRate = (_amount + leftover) / rewardsDuration;
    }
```

---

## Impact Explanation

**High Impact**

The impact can be diferent in diferent scenarios. For example the more there are no stakers the more rewards will be locked. 

---

## Likelihood Explanation

**High Likelihood**

* There is no function or a way to retrieve or redistribute the unalocated rewards. And this can hapend evry time `block.timestamp >= periodFinish` and a new period is started.

---

## Proof of Concept

The test simulates two users: one who stakes once and holds their position, and another who stakes and exits repeatedly, accelerating reward calculations. It confirms that rewardPerToken accumulates rewards proportionally and precisely over time, ensuring the final reward transfer matches the expected earned amount for the long-term staker. 

However, before the users start stakeing into the contract there is a period in which nobody is staking, during that time rewards are not going to be accumulated and eventually locked. There is no way to retrieve these funds. 
```
├─ emit log_named_uint(key: "Staker (The contract in which funds are stuck) reward token balance after period finish", val: 273972602757232501 [2.739e17])
├─ emit log_named_uint(key: "Investor reward token balance after exit", val: 684931506832799999 [6.849e17])
├─ emit log_named_uint(key: "Investor2 reward token balance after exit", val: 41095890409967500 [4.109e16])
```
- Add this function inside `Staker.t.sol` and run it with `forge test --mt test_Reward_Getting_Stuck_In_The_Contract_Over_Time -vvvv`
```solidity
    // Tests if rewardPerToken works correctly
    function test_Reward_Getting_Stuck_In_The_Contract_Over_Time(
        uint256 investment
    ) public {
        vm.assume(investment != 0 && investment < 1e25);
        
        address investor = vm.addr(uint256(keccak256(bytes("Investor"))));
        address investor2 = vm.addr(uint256(keccak256(bytes("Investor2"))));
        deal(tokenIn, investor, investment);
        deal(tokenIn, investor2, investment * 10);

        mintRewardsAndApprove(OWNER, 1e18);

        vm.prank(OWNER, OWNER);
        staker.addRewards(OWNER, 1e18);

        //Time passes without any user activity
        // I know 100 days is a lot but it is just for the example to show that the rewards are getting stuck
        vm.warp(block.timestamp + 100 days);

        // investor1 makes a deposit from which he will claim rewards at the end
        vm.startPrank(investor, investor);
        IERC20Metadata(tokenIn).approve(address(staker), investment);
        staker.deposit(investment);
        vm.stopPrank();

        // investor2 deposits and exits 500 times to claim rewards
        // this simulates normal user activity
        for (uint256 i = 0; i < 500; i++) {
            vm.startPrank(investor2, investor2);
            IERC20Metadata(tokenIn).approve(address(staker), investment);
            staker.deposit(investment);
            vm.warp(block.timestamp + 0.6 days);
            staker.exit();
            vm.stopPrank();
        }
        
        uint256 periodFinish = staker.periodFinish();
        
        // We fast forward to the end of the reward period
        uint256 warpAmount = staker.periodFinish() - block.timestamp + 100 days;
        vm.warp(block.timestamp + warpAmount);

        // Log reward state before exit
        uint256 rewardBalanceBefore = IERC20Metadata(rewardToken).balanceOf(address(staker));
        uint256 investorRewardBefore = IERC20Metadata(rewardToken).balanceOf(investor);
        uint256 earnedBefore = staker.earned(investor);
        
        emit log_named_uint("Period finish", periodFinish);
        emit log_named_uint("Current timestamp", block.timestamp);
        emit log_named_uint("Last time reward applicable", staker.lastTimeRewardApplicable());
        
        // Investor1 exit
        vm.prank(investor, investor);
        staker.exit();
        
        // Log reward state after exit
        uint256 rewardBalanceAfter = IERC20Metadata(rewardToken).balanceOf(address(staker)); // Leftover rewards from the period
        uint256 investorRewardAfter = IERC20Metadata(rewardToken).balanceOf(investor);
        uint256 investor2RewardAfter = IERC20Metadata(rewardToken).balanceOf(investor2);
        
        emit log_named_uint("Staker (The contract in which funds are stuck) reward token balance after period finish", rewardBalanceAfter);
        emit log_named_uint("Investor reward token balance after exit", investorRewardAfter);
        emit log_named_uint("Investor2 reward token balance after exit", investor2RewardAfter);
        
        // Verify rewards were transferred correctly
        assertEq(investorRewardAfter - investorRewardBefore, earnedBefore, "Investor didn't receive correct rewards");
        assertEq(rewardBalanceBefore - rewardBalanceAfter, earnedBefore, "Staker didn't transfer correct rewards");
        
        // Verify investor's state after exit
        assertEq(staker.balanceOf(investor), 0, "Investor's balance after exit should be 0");
        assertEq(staker.rewards(investor), 0, "Investor's rewards after exit should be 0");
    }
```

---

## Recommendation

Enhance the `addRewards()` to add the leftover rewards in the contract to the newly transfered rewards for the next period.

```diff
    function addRewards(
        address _from,
        uint256 _amount
    ) external override onlyOwner validAmount(_amount) updateReward(address(0)) {
        // Transfer assets from the `_from`'s address to this contract.
        IERC20(rewardToken).safeTransferFrom({ from: _from, to: address(this), value: _amount });

        require(rewardsDuration > 0, "3089");
        if (block.timestamp >= periodFinish) {
-           rewardRate = _amount / rewardsDuration;
+           rewardRate = _amount + IERC20(rewardToken).balanceOf(address(this)) / rewardsDuration;
        } else {
            uint256 remaining = periodFinish - block.timestamp;
            uint256 leftover = remaining * rewardRate;
            rewardRate = (_amount + leftover) / rewardsDuration;
        }
 
        // Prevent setting rewardRate to 0 because of precision loss.
        require(rewardRate != 0, "3088");

        // Prevent overflows.
        uint256 balance = IERC20(rewardToken).balanceOf(address(this));
        require(rewardRate <= (balance / rewardsDuration), "2003");

        lastUpdateTime = block.timestamp;
        periodFinish = block.timestamp + rewardsDuration;
        emit RewardAdded(_amount);
    }
```

Or implement a function to recover ERC20 tokens that might have been accidentally or otherwise left within the contract.

