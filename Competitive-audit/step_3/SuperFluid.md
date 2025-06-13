**Title: SuperFluid**

Auditor: **vesko210**

## Issues found
|Severtity|Number of issues found
| ------- | -------------------- |
| High    | 2                    |
| Medium  | 0                    |
| Low     | 1                    |

# Findings

## [H-1] Full‑Range Uniswap V3 positions lead to negligible fees and high impermanent loss

## Summary  
The locker’s `provideLiquidity` function always mints a full‑range Uniswap V3 position—from `TickMath.MIN_TICK` to `TickMath.MAX_TICK`. While this is functionally valid, it severely undermines capital efficiency and significantly reduces fee-earning potential.

## Finding Description  
In `_createPosition(...)`, the contract hard-codes the following tick bounds:

```solidity
tickLower: (TickMath.MIN_TICK / tickSpacing) * tickSpacing,
tickUpper: (TickMath.MAX_TICK / tickSpacing) * tickSpacing,
```

This creates a position that spans the entire allowable price range of the pool (e.g., ETH/SUP). Unlike concentrated liquidity positions, which strategically place capital around active price zones, full-range positions spread liquidity evenly—even into price regions that are never reached.

## Impact Explanation

### Fee Revenue Loss

Full‑range LP positions earn very little in swap fees during normal market activity. Since most trades occur within a narrow price band, wide liquidity placement results in significantly diminished yield potential.

### User Capital Misallocation

Users’ capital is deployed in ranges where trading rarely, if ever, occurs. This inefficient allocation weakens market depth in active price zones and reduces capital productivity.

## Likelihood Explanation

This outcome is certain due to the code’s hard-coded behavior. Every invocation of `provideLiquidity` results in a full-range position. There is no path for user input or dynamic logic to alter the tick range.

## Recommendation

Avoid hardcoding full-range tick values. Instead:

### Allow Configurable Tick Ranges

Let users pass in `tickLower` and `tickUpper` as parameters to `provideLiquidity`. This gives experienced users control over capital placement and optimizes yield potential.


## [H-2] Insufficient LP_OPERATION_DEADLINE exposes liquidity functions to DoS in high volatile times

## Summary 
The hard‑coded 1‑minute `LP_OPERATION_DEADLINE` for Uniswap V3 mint/decrease calls can easily expire under normal network conditions, causing transaction reverts and denial of service for liquidity operations.

## Finding Description  
In both `_createPosition` and `_decreasePosition`, the contract uses:

```solidity
deadline: block.timestamp + LP_OPERATION_DEADLINE
```

where:

```solidity
uint256 public constant LP_OPERATION_DEADLINE = 1 minutes;
```

Uniswap V3 will revert if the current block timestamp exceeds the provided deadline. Across multiple target chains—each with varying block times, mempool delays, and potential MEV relay queues—a 60-second window is frequently insufficient. This problem is amplified in multi-chain deployments, especially on volatile or congested networks.

Uniswap’s official interface typically defaults to a much longer deadline (e.g., 20–30 minutes), acknowledging these risks. Given that users already have slippage protection, increasing the deadline does not introduce meaningful additional risk.

## Failure Propagation

1. User calls `provideLiquidity` or `withdrawLiquidity`.
2. The transaction enters the mempool.
3. Network congestion, gas auctions, or relay delays prevent inclusion within 60 seconds.
4. `block.timestamp > deadline` → Uniswap reverts → entire Locker transaction fails.
5. User loses gas and receives no actionable feedback.

## Impact Explanation

* **Availability:** Routine liquidity provision or withdrawal can unexpectedly fail, blocking user actions.
* **User Frustration:** Users may waste time and gas adjusting transaction parameters to fit within the tight deadline.
* **Cost:** In high congestion scenarios, users are forced to overpay for gas just to meet a strict timing window.

## Recommendation

The goal is to give user transactions more time to be processed without introducing risk, as slippage protection still applies.

### Option A: Increase the Constant

Increase the deadline to match Uniswap frontend defaults:

```diff
- uint256 public constant LP_OPERATION_DEADLINE = 1 minutes;
+ uint256 public constant LP_OPERATION_DEADLINE = 20 minutes;
```

### Option B: Make Deadline Configurable

Allow users to define a custom deadline (within a safe range, e.g., 1–20 minutes) when initiating liquidity operations, increasing flexibility across various network conditions.


## [L-1] Unconditional disconnection on partial unstake halts rewards

## Summary 
Unstaking any amount—no matter how small—always calls `disconnectPool(...)`, fully severing the locker’s connection to the staker distribution pool even if some stake remains.

## Finding Description  
In `FluidLocker.unstake(uint256 amountToUnstake)`, after reducing `_stakedBalance` and updating the pool’s units, the code always executes:

```solidity
// Disconnect this locker from the Tax Distribution Pool
FLUID.disconnectPool(STAKER_DISTRIBUTION_POOL);
```

Whether the user unstakes 0.1 FLUID or their entire balance, they are disconnected, so their remaining stake no longer accrues rewards. They must then call `connectToPool` or `stake(...)` to reconnect.

## Impact Explanation

### User Experience & Funds

Users who attempt to partially unstake will unexpectedly stop earning rewards on their remaining balance.

## Likelihood Explanation

**Very likely** in normal usage:

1. A user stakes 10 FLUID and waits out the cooldown.
2. They decide to free up 1 FLUID of liquidity.
3. Calling `unstake(1e18)` forcibly disconnects—even though 9 FLUID remains staked.
4. To resume earning on the 9 FLUID, they must call `stake(9e18)` → triggers a new 3‑day cooldown or `connectToPool`.

Because partial unstakes are common, this will almost always impact users.

## Recommendation

Only disconnect when the user’s entire staked balance goes below the minimum for a whole Unit. Change `unstake()` as follows:

```diff
     // Update the staked balance
     _stakedBalance -= amountToUnstake;

     // Call Staking Reward Controller to update staker's units
     STAKING_REWARD_CONTROLLER.updateStakerUnits(_stakedBalance);

-    // Disconnect this locker from the Tax Distribution Pool
-    FLUID.disconnectPool(STAKER_DISTRIBUTION_POOL);
+    // Only disconnect when no units remain staked
+    if (_stakedBalance < 1e18) {
+        FLUID.disconnectPool(STAKER_DISTRIBUTION_POOL);
     }
```
