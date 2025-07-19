**Title: succinct-network**

Auditor: **vesko210**

## Issues found
|Severtity|Number of issues found
| -------- | -------------------- |
| High     | 1                    |
| Med      | 3                    |
| Info     | 1                    |

# Findings

# [H-1] Stakers still receive dispenced rewards even after unstakeRequest

## Summary

Stakers continue to accrue and receive new rewards via `dispense()` even after they have submitted an `unstakeRequest`, meaning they are still treated as active stakers during the unstaking delay period. They cannot earn rewards from the prover but are still able to receive rewards from `dispense()`.

## Finding Description

When a staker calls `requestUnstake(stPROVE)`, their stake tokens (`stPROVE`) remain in the pool for reward distribution. The contract does not remove or “freeze” that staker’s balance from subsequent `dispense()` calls until `finishUnstake()` is executed. As a result, between the timestamp of `requestUnstake` and `finishUnstake`, the staker still receives new PROVE rewards.

Once a user signals intent to withdraw, they should stop earning newly dispensed rewards on that portion of their stake. Malicious or impatient users can call `requestUnstake`, wait the full unstake delay, then still collect a share of all dispensed rewards before finally calling `finishUnstake`. This way they can defend themselves from slashes and still receive dispense rewards.

## Impact Explanation

* **Impact Level**: High

Because pending‑unstakes are still eligible for rewards, users can stake into a prover and create an `unstakeRequest`. After the unstake period passes, they remain eligible for dispensed rewards and can call `finishUnstake` right before a `requestSlash` is created—avoiding slashing losses while still collecting rewards.

## Likelihood Explanation

* **Likelihood**: High

The behavior requires no unusual conditions.

## Proof of Concept

Add the following import to your test file:

```solidity
import "lib/forge-std/src/console.sol";
```

Run the test: `forge test --mt test_Unstake_StakerStillRecievesRewards_EvenAfter_RequestUnstake -vvvv`

```solidity
function test_Unstake_StakerStillRecievesRewards_EvenAfter_RequestUnstake() public {
    uint256 stakeAmount = STAKER_PROVE_AMOUNT;
    uint256 dispenseAmount = STAKER_PROVE_AMOUNT / 2;

    console.log(IERC20(PROVE).balanceOf(STAKER_1));

    uint256 amount1 = _stake(STAKER_1, ALICE_PROVER, stakeAmount);

    // Unstake from Alice prover
    _requestUnstake(STAKER_1, amount1);

    uint256 requiredTime = dispenseAmount / DISPENSE_RATE + 1;
    skip(requiredTime);

    vm.prank(DISPENSER);
    SuccinctStaking(STAKING).dispense(dispenseAmount);

    _finishUnstake(STAKER_1);

    console.log(IERC20(PROVE).balanceOf(STAKER_1));
}
```

**Output:**

```solidity
[PASS] test_Unstake_StakerStillRecievesRewards_EvenAfter_RequestUnstake() (gas: 506322)
Logs:
  100000000000000000000000
  149999999999999999999999
```

## Recommendation

When an `unstakeRequest` is created, redeem the underlying `stPROVE` immediately, sending it into the contract so it is removed from the reward pool. Later, when the user calls `finishUnstake`, send them the PROVE amount minus any slashed share (if applicable) corresponding to their redeemed portion.


# [M-1] Excess returned to the prover in `_unstake` can be front-run

## Summary

When `finishUnstake` is called, any excess rewards (i.e., the difference between the `iPROVE` received and the snapshot) are sent back to the `_prover`. This can be front‑run, allowing an attacker to steal rewards that would otherwise be distributed to honest stakers.

## Finding Description

When a `requestUnstake` is created, a snapshot of how much `iPROVE` is expected to be redeemed is taken. If during the `unstakePeriod` there is a reward distribution, then at `finishUnstake` the user will have earned additional `iPROVE`. The excess beyond the snapshot is returned to the prover, boosting everyone’s balance in the prover pool. An attacker can front‑run a large unstake to capture this excess.

```solidity
/// @dev Burn a staker's $stPROVE and withdraw $PROVER-N to receive $iPROVE, then withdraw
///      $iPROVE to receive $PROVE, which gets sent to the staker.
///
///      The actual $iPROVE withdrawn is adjusted to ensure that prover rewards earned during the
///      unstaking period do not go to the staker, while ensuring that slashing that occurs during
///      the unstaking period does. This is implemented as follows:
///      - If received $iPROVE > snapshot $iPROVE: Rewards were earned during unstaking, so the
///        staker redeems only the snapshot amount, and the excess is returned to the prover.
///      - If received $iPROVE < snapshot $iPROVE: Slashing occurred during unstaking, so the
///        staker redeems the lower amount (received $iPROVE), bearing the loss from slashing.
function _unstake(
    address _staker,
    address _prover,
    uint256 _stPROVE,
    uint256 _iPROVESnapshot
) internal returns (uint256 PROVE) {
    // Ensure unstaking a non-zero amount.
    if (_stPROVE == 0) revert ZeroAmount();

    // Burn the $stPROVE from the staker
    _burn(_staker, _stPROVE);

    // Withdraw $PROVER-N from this contract to have this contract receive $iPROVE.
    uint256 iPROVEReceived = IERC4626(_prover).redeem(_stPROVE, address(this), address(this));

    // Determine how much $iPROVE to redeem for the staker based on rewards or slashing.
    uint256 iPROVE;
    if (iPROVEReceived > _iPROVESnapshot) {
        // Rewards were earned during unstaking. Return the excess to the prover.
        uint256 excess = iPROVEReceived - _iPROVESnapshot;
        IERC20(iProve).safeTransfer(_prover, excess);
        iPROVE = _iPROVESnapshot;
    } else {
        // Either no change or slashing occurred. Staker gets what's available.
        iPROVE = iPROVEReceived;
    }

    // Withdraw $iPROVE from this contract to have the staker receive $PROVE.
    PROVE = IERC4626(iProve).redeem(iPROVE, _staker, address(this));

    emit Unstake(_staker, _prover, PROVE, iPROVE, _stPROVE);
}
```

## Impact Explanation

* **Impact Level**: Medium
* **Risk**: Allows a front‑runner to steal rewards that would otherwise be distributed to honest stakers by capturing the excess `iPROVE` returned to the prover.

## Likelihood Explanation

No special conditions are required. An attacker merely needs to detect a pending unstake transaction of sufficient size and front‑run it to profit from the returned excess.

## Proof of Concept

Add the following import to your test file:

```solidity
import "lib/forge-std/src/console.sol";
```

Run the test: `forge test --mt test_Unstake_WhenTwoStakers_Dispense_When_DispenseRate_Changes -vvvv`

```solidity
function testFuzz_Reward_WhenWithPartialUnstake_FrontRun(
    uint256 _unstakePercent,
    uint256 _rewardAmount
) public {
    uint256 stakeAmount = STAKER_PROVE_AMOUNT;
    uint256 unstakePercent = bound(_unstakePercent, 10, 90); // 10–90% unstake
    uint256 unstakeAmount = (stakeAmount * unstakePercent) / 100;
    uint256 rewardAmount = bound(
        _rewardAmount,
        REQUESTER_PROVE_AMOUNT / 10,
        REQUESTER_PROVE_AMOUNT / 4
    );

    // Stake
    _stake(STAKER_1, ALICE_PROVER, stakeAmount);

    // Request partial unstake
    _requestUnstake(STAKER_1, unstakeAmount);

    // Process reward while unstaking
    MockVApp(VAPP).processFulfillment(ALICE_PROVER, rewardAmount);

    // Calculate and withdraw
    (uint256 protocolFee, uint256 stakerReward, uint256 ownerReward) =
        _calculateFullRewardSplit(rewardAmount);
    _withdrawFromVApp(FEE_VAULT, protocolFee);
    _withdrawFromVApp(ALICE, ownerReward);
    _withdrawFromVApp(ALICE_PROVER, stakerReward);

    // Complete unstake
    skip(UNSTAKE_PERIOD);

    uint256 stake2 = _stake(STAKER_2, ALICE_PROVER, stakeAmount);

    uint256 receivedAmount1 = _finishUnstake(STAKER_1);

    _requestUnstake(STAKER_2, stake2);

    skip(UNSTAKE_PERIOD);

    uint256 receivedAmount2 = _finishUnstake(STAKER_2);

    // Verify unstaked amount doesn’t include new rewards
    assertApproxEqAbs(receivedAmount1, unstakeAmount, 1);

    console.log(receivedAmount2);
}
```

## Recommendation

Instead of returning excess `iPROVE` directly to the prover, send it back into the contract so the Dispenser can later redistribute these funds to honest stakers.


# [M-2] Stakers exposed to unexpected losses via changing dispense rate

## Summary

Stakers continue to accrue and receive new rewards via `dispense()` even after they have submitted an `unstakeRequest`, meaning they are still treated as active stakers during the unstaking delay period. They cannot earn rewards from the prover but are still able to receive rewards from `dispense()`.

## Finding Description

When a staker calls `requestUnstake(stPROVE)`, their stake tokens (`stPROVE`) remain in the pool for reward distribution. The contract does not remove or “freeze” that staker’s balance from subsequent `dispense()` calls until `finishUnstake()` is executed. As a result, between the timestamp of `requestUnstake` and `finishUnstake`, the staker still receives new PROVE rewards.

Once a user signals intent to withdraw, they should stop earning newly dispensed rewards on that portion of their stake. Malicious or impatient users can call `requestUnstake`, wait the full unstake delay, then still collect a share of all dispensed rewards before finally calling `finishUnstake`. This way they can defend themselves from slashes and still receive dispense rewards.

## Impact Explanation

* **Impact Level**: High

Because pending‑unstakes are still eligible for rewards, users can stake into a prover and create an `unstakeRequest`. After the unstake period passes, they remain eligible for dispensed rewards and can call `finishUnstake` right before a `requestSlash` is created—avoiding slashing losses while still collecting rewards.

## Likelihood Explanation

* **Likelihood**: High

The behavior requires no unusual conditions.

## Proof of Concept

Add the following import to your test file:

```solidity
import "lib/forge-std/src/console.sol";
```

Run the test: `forge test --mt test_Unstake_StakerStillRecievesRewards_EvenAfter_RequestUnstake -vvvv`

```solidity
function test_Unstake_StakerStillRecievesRewards_EvenAfter_RequestUnstake() public {
    uint256 stakeAmount = STAKER_PROVE_AMOUNT;
    uint256 dispenseAmount = STAKER_PROVE_AMOUNT / 2;

    console.log(IERC20(PROVE).balanceOf(STAKER_1));

    uint256 amount1 = _stake(STAKER_1, ALICE_PROVER, stakeAmount);

    // Unstake from Alice prover
    _requestUnstake(STAKER_1, amount1);

    uint256 requiredTime = dispenseAmount / DISPENSE_RATE + 1;
    skip(requiredTime);

    vm.prank(DISPENSER);
    SuccinctStaking(STAKING).dispense(dispenseAmount);

    _finishUnstake(STAKER_1);

    console.log(IERC20(PROVE).balanceOf(STAKER_1));
}
```

**Output:**

```solidity
[PASS] test_Unstake_StakerStillRecievesRewards_EvenAfter_RequestUnstake() (gas: 506322)
Logs:
  100000000000000000000000
  149999999999999999999999
```

## Recommendation

When an `unstakeRequest` is created, redeem the underlying `stPROVE` immediately, sending it into the contract so it is removed from the reward pool. Later, when the user calls `finishUnstake`, send them the PROVE amount minus any slashed share (if applicable) corresponding to their redeemed portion.


# [M-3] Prover Ownership Immutability Vulnerability

## Summary

Prover ownership cannot be transferred. Once deployed, the `owner` of a Prover contract is immutable, preventing any change of the prover operator.

## Finding Description

There is no `transferOwnership` or similar mechanism in the Prover contract. This lack of on‑chain recovery prevents any path to rotate or change the deploying key for scenarios such as:

* Inability to continue operating the node (e.g., key loss)
* Selling or delegating the prover
* Owner address blacklisted from the underlying fee token
* Compromised owner key

If any of these occur, the prover will cease operation since the owner cannot claim fees or perform administrative functions. Stakers’ funds could be at risk if the prover is slashed or abandoned due to an unmaintained or inaccessible owner key.

## Impact Explanation

* **Impact Level:** Medium

  * Inability to rotate or change ownership reduces operational resilience and flexibility. In production, stakers could suffer financial losses if prover operations halt.

## Likelihood Explanation

* **Likelihood:** Low

  * This scenario requires one of the following: node operator key loss, sale/delegation needs, blacklisting of owner address, or key compromise.

## Proof of Concept

The `owner` is declared immutable with no setter:

```solidity
contract SuccinctProver is ERC4626, IProver {
    // ...
    /// @inheritdoc IProver
    address public immutable override owner;
    // ...
}
```

There is no function in the Prover contract or interface to update `owner`. In tests:

```solidity
function test_CreateProver_CanNot_Transfer_Ownership() public {
    address proverOwner = makeAddr("PROVER_OWNER");

    vm.prank(proverOwner);
    address prover = SuccinctStaking(STAKING).createProver(STAKER_FEE_BIPS);

    assertEq(IProver(prover).owner(), proverOwner);
    assertEq(IProver(prover).id(), 3);
    assertEq(ERC20(prover).name(), "SuccinctProver-3");
    assertEq(ERC20(prover).symbol(), "PROVER-3");

    // THERE IS SIMPLY NO FUNCTION TO CHANGE THE OWNER
}
```

## Recommendation

Make the `owner` mutable and implement a standard `transferOwnership` function (e.g., OpenZeppelin’s `Ownable`) to allow secure on‑chain ownership transfers and key rotations.


# [I-1] Prover Owner Fee Omission Vulnerability

## Summary

Prover owners never receive the `_stakerFeeBips` fee they specify at prover creation, because the staking logic never transfers any portion of the stake to the prover owner.

## Finding Description

When a new prover is created, the protocol records a per‑prover `stakerFeeBips` (basis points) value intended to compensate the prover owner on each stake. However, throughout the `stake`, `permitAndStake`, `requestUnstake`, `finishUnstake`, and related internal flows in `SuccinctStaking`, this fee value is never used:

* No calculation is performed to split a user’s stake between “fee” and “net stake.”
* No transfer of fee to the prover owner is executed.
* All deposited PROVE flows directly into the iPROVE vault, with 0% diverted.

This breaks the economic guarantee that prover owners earn a configurable cut of the stakes they secure.

## Impact Explanation

* **Impact Level:** Medium

  * Critically undermines economic incentives for running provers. Without compensation, prover operators lack motivation to onboard or maintain nodes. Network growth may stall because operators see no return on their collateral or operational costs.

## Likelihood Explanation

* **Likelihood:** High

  * Every prover creation includes a non‑zero `stakerFeeBips` in practice. Because the logic never references it, all provers in production will suffer from zero owner compensation.

## Proof of Concept

In the following test, it’s clear that neither the prover contract nor its owner receive any tokens when `STAKER_1` stakes, rewards are dispensed, and then unstakes:

Add this test inside of `SuccinctStakingDispenseTests` and run: `forge test --mt test_Prover_Owner_Does_Not_Recieve_Fees -vvvv`

```solidity
function test_Prover_Owner_Does_Not_Recieve_Fees() public {
    uint256 stakeAmount = STAKER_PROVE_AMOUNT;
    uint256 dispenseAmount = STAKER_PROVE_AMOUNT;

    // Stake to Alice prover
    _stake(STAKER_1, ALICE_PROVER, stakeAmount);

    // Check state after staking
    assertEq(SuccinctStaking(STAKING).balanceOf(STAKER_1), stakeAmount);
    assertEq(SuccinctStaking(STAKING).staked(STAKER_1), stakeAmount);

    // Dispense
    _dispense(dispenseAmount);

    // Verify rewards were distributed
    assertEq(SuccinctStaking(STAKING).balanceOf(STAKER_1), stakeAmount);
    uint256 expectedStakedAfterDispense = stakeAmount + dispenseAmount;
    uint256 actualStaked = SuccinctStaking(STAKING).staked(STAKER_1);
    assertApproxEqAbs(actualStaked, expectedStakedAfterDispense, 10);
    assertEq(IERC20(PROVE).balanceOf(I_PROVE), stakeAmount + dispenseAmount);

    // Complete unstake process
    _completeUnstake(STAKER_1, stakeAmount);

    // Verify final state
    assertEq(SuccinctStaking(STAKING).balanceOf(STAKER_1), 0);
    assertEq(SuccinctStaking(STAKING).staked(STAKER_1), 0);

    // All PROVE was sent to the staker
    assertApproxEqAbs(IERC20(PROVE).balanceOf(STAKER_1), stakeAmount + dispenseAmount, 10);
    assertApproxEqAbs(IERC20(PROVE).balanceOf(I_PROVE), 0, 10);
    assertEq(IERC20(I_PROVE).balanceOf(ALICE_PROVER), 0);

    // Verify protocol fees were not transferred to FEE_VAULT
    assertEq(
        IERC20(PROVE).balanceOf(FEE_VAULT),
        0,
        "Protocol fees were not transferred to FEE_VAULT"
    );

    // Verify owner rewards were not transferred to prover owner (ALICE)
    assertEq(
        IERC20(PROVE).balanceOf(ALICE),
        0,
        "Owner rewards were not transferred to prover owner"
    );
}
```

## Recommendation

Incorporate `stakerFeeBips` into the staking flow.