**Title: Alchemix**

Auditor: **vesko210**

## Issues found
|Severtity|Number of issues found
| ------- | -------------------- |
| High    | 3                    |

# Findings

# [H-1] Unallocated rewards get permanently stuck when no stakers are present during reward period

## Summary

Unallocated reward tokens can become permanently stuck in the `GaugeV2` contract if a reward period starts while no users have stakeed.

---

## Finding Description

The `notifyRewardAmount()` function in the `GaugeV2` contract initializes a new reward period by setting the `rewardRate` and `_periodFinish`. However, if there are no stakers `_totalSupply == 0` and the period has started, for every second that passes rewards will be lost at the `rewardRate`. When the reward period ends, any unallocated rewards remain in the contract and are not reused in subsequent periods.

A portion of the funds becomes **inaccessible**, resulting in inefficient fund usage and potential monetary loss.

This issue arises passively when `notifyRewardAmount()` is called with a valid reward amount, but there are no stakers at the start of the period, and the more there are no stakers the more reward tokens will be lost.

Relevant code: 
Unalocated tokens from the begining of the period are not added to the rewardRate.
```solidity
if (block.timestamp >= _periodFinish) {
    // NOTE if there are left over funds for the past period, the reward tokens will be stuck
@>  rewardRate = reward / DURATION;
} else {
    uint256 remaining = _periodFinish - block.timestamp;
    uint256 leftover = remaining * rewardRate;
@>  rewardRate = (reward + leftover) / DURATION;
}
```
---

## Impact Explanation

**High Impact**

The issue results in the **permanent loss** of rewards that are transferred to the contract.

---

## Likelihood Explanation

**High Likelihood**

This issue can occur naturally in realistic scenarios.
No malicious actor is required for this to occur, though it can be deliberately triggered.

---

## Proof of Concept

### Step-by-Step Reproduction:

1. Deploy the `GaugeV2` contract.
2. No users have staked.
3. The `DISTRIBUTION` address, calls `notifyRewardAmount` and adds 100 ether
4. For the time that there are no stakers and the reward period is going, 
   Funds will be lost: `rewardRate * block.timestamp - StartPeriod`.
   Lets say half the time there are no stakers, this would mean 50 ether permanently lost.
5. Wait for the duration to pass (default 1 week).
6. After the period ends, call `notifyRewardAmount()` again to start a new reward cycle.
7. Observe that:

   * The not distributed 50 tokens are not included in the new reward rate.
   * The contract holds those 50 tokens with no way to recover or reallocate them.

---

## Recommendation

Implement logic to **handle unallocated rewards** when a new reward is notified. One solution is to carry over the unused reward tokens from the previous/current period if they were not distributed.


# [H-2] Governance emission adjustment ignored when weekly emission above tail threshold

## Summary

The protocol only applies the DAO-controlled `tailEmissionRate` when the weekly emission drops below `TAIL_START`. This prevents governance from influencing emissions during the intended phase (epoch ≥ 67), which contradicts the documented behavior and severely limits DAO control.

---

## Finding Description

In the current implementation of `update_period()`, the weekly emission is multiplied by `tailEmissionRate` **only when** `_weekly < TAIL_START`. This logic is meant to apply the DAO's voted emission adjustment in the governance-controlled phase (from epoch 67 onwards). However, this condition prevents DAO influence if emissions remain above `TAIL_START`, effectively locking the emission rate even after governance takes over.

This violates the protocol’s documented guarantee:

> "On and after the 67th epoch, it relies on the Governance DAO. If the DAO votes in favor, it increases by 1% of the previous epoch’s emission; if the proposal is against, emission will decrease by 1%; else it will remain the same."

Because the DAO's control is blocked when `_weekly >= TAIL_START`, the contract fails to fulfill its core governance mechanism — undermining decentralization, community control, and emission responsiveness.

---

## Impact Explanation

The impact is **High**. The protocol enters a governance phase after epoch 66, but the DAO’s vote is ignored unless emissions are below a fixed threshold. This means DAO proposals may appear to pass, but have no effect — creating a false sense of control. It limits emission modulation, disrupts monetary policy, and could erode community trust.

---

## Recommendation

Remove the `if (_weekly < TAIL_START)` conditional and apply `tailEmissionRate` unconditionally when `epochCount >= 67`, as per the documentation.

### Recommended Fix:

```solidity
    if (block.timestamp >= _period + WEEK && _initializer == address(0)) {
        epochCount++;
        _period = (block.timestamp / WEEK) * WEEK;
        active_period = _period;

        uint256 _weekly = weekly;
        uint256 _emission;

        // Phase 1: Epochs 1–14 (inclusive) — 3% growth
        if (epochCount < 15) {
            _weekly = (_weekly * WEEKLY_GROWTH) / MAX_BPS;

        // Phase 2: Epochs 15–66 (inclusive) — 1% growth
        } else if (epochCount < 67) {
            _weekly = (_weekly * 10100) / MAX_BPS;

        // Phase 3: Epochs 67+ — DAO governance control via nudge()
        } else {
            // Apply governance-controlled tailEmissionRate from prior vote
            // Note: tailEmissionRate will have been set via `nudge()`
            _weekly = (_weekly * tailEmissionRate) / MAX_BPS;
        }
```

This allows the DAO to truly control emissions starting at epoch 67, as promised in the docs.


# [H-1] Emission schedule deviation: 1% weekly decrease instead of increase

## Summary

The contract emits **decreasing weekly rewards (−1%)** during epochs 15–66, while the documentation specifies a **+1% increase** during this period. This results in a significant under-distribution of tokens and deviates from the publicly stated tokenomics.

---

## Finding Description

The `update_period()` function within the `MinterUpgradeable` contract handles the emission schedule. According to the documented behavior:

> “Start emission 10M in the first epoch, after that for the next 14 epochs it will increase by 3%, then it’ll increase by 1% for the next 52 epochs.”

However, the implementation mistakenly applies a **1% weekly decay** instead of a 1% growth from **epoch 15 to epoch 66**:

```solidity
if (epochCount < 15) {
    _weekly = (_weekly * WEEKLY_GROWTH) / MAX_BPS;
} else {
    _weekly = (_weekly * WEEKLY_DECAY) / MAX_BPS; // 9900 → -1% decay
}
```

Here, `WEEKLY_DECAY` is set to `9_900`, causing the weekly emission to shrink by 1% per epoch rather than grow by 1% as expected. This is a **functional deviation from the defined economic model.**

---

## Impact Explanation

**Impact: High**

This bug results in a **\~64% reduction** in emissions compared to what the documentation promises for epochs 15–66. Instead of increasing the token supply, the protocol *deflates* emissions — reducing yield, rewards.

---

## Likelihood Explanation

**Likelihood: High**

The bug is deterministic and occurs automatically when `epochCount >= 15`. No external input or attacker action is required — this logic flaw is **inevitably triggered** in normal operation of the protocol.

---

## Proof of Concept

That is just un example of that how much the diference is going to be:

```solidity
// Set epochCount = 15 to simulate start of second phase
weekly = 10_000_000e18;
WEEKLY_DECAY = 9900; // 1% decay
MAX_BPS = 10_000;

// Simulate weekly update logic
weekly = (weekly * WEEKLY_DECAY) / MAX_BPS; 
// Result: 9,900,000 BLACK instead of 10,100,000

// Expected:
weekly = (weekly * 10_100) / MAX_BPS; 
// Result: 10,100,000 BLACK
```

After 52 epochs:

```text
Expected: weekly = 10M * (1.01)^52 ≈ 16.7M BLACK
Actual:   weekly = 10M * (0.99)^52 ≈ 6.0M BLACK
```
16.77M / 6.04M = 0.36
So the actual emission is only 36% of what it should be.
This 64% difference reflects a large under-distribution of tokens against the intended behavior.

---

## Recommendation

Replace the `WEEKLY_DECAY` logic for epochs 15–66 with a **growth multiplier** matching the documentation (1% increase).

### Fixed Code Snippet:

Replace:

```diff
+ uint256 public constant WEEKLY_GROWTH_LATE = 10_100; // +1%

// code...

if (epochCount < 15) {
    _weekly = (_weekly * WEEKLY_GROWTH) / MAX_BPS;
} else (epochCount < 67) {
+   _weekly = (_weekly * WEEKLY_GROWTH_LATE) / MAX_BPS;
}
```