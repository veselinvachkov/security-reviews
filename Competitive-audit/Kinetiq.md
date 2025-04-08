**Title: Kinetiq**

Auditor: **vesko210**

## Issues found
|Severtity|Number of issues found
| ------- | -------------------- |
| Medium  | 2                    |

# Findings

## [M-1]Whitelist can cause disabled withdrawal function for some users

## Finding description and impact
Setting whitelist Enabled to true during a normal operation of the contract will lead to some users losing the ability to withdrawal their funds. While the whitelist feature is intended to control who can stake and withdraw from the protocol, it lacks proper protections for users who enter the system while the whitelist is disabled but later find themselves restricted due to a change in `whitelistEnabled` state.

More specifically, the `stake()` function allows users to deposit funds into the contract and receive `kHYPE` tokens when the whitelist is turned off. However, at any point after staking, the whitelist can be re-enabled. When this occurs, users who are not on the whitelist are locked out from initiating a withdrawal.

The relevant check can be seen in the `queueWithdrawal()` function:

```solidity
function queueWithdrawal(uint256 kHYPEAmount) external nonReentrant whenNotPaused whenWithdrawalNotPaused {
        // Check whitelist if enabled
        if (whitelistEnabled) {
            require(isWhitelisted(msg.sender), "Address not whitelisted");
        }
        // code...
}
```

- Users may lose timely access to their staked funds.
- Users may need their funds as soon as possible in an emergency that may be blocked by the whitelist, and the user will have to wait for the contract to set the whitelist to disabled to request their funds.

Also the code comments over the `enableWhitelist` and `disableWhitelist` mentions that but overall it does not account that the whitelist is also for withdrawal, so i think that it is not ment to stop users from withdrawing.
```solidity
    /**
     * @notice Disable whitelist for staking
     */
```

## PoC:
You can add this test into the `StakingManager.t.sol` and run it with:
- forge test --mt test_Withdrawal_When_Whitelist_Enabled -vvv

```solidity
function test_Withdrawal_When_Whitelist_Enabled() public {
        uint256 stakeAmount = 1 ether;

        // Set up delegation first
        vm.startPrank(manager);
        validatorManager.activateValidator(validator);
        validatorManager.setDelegation(address(stakingManager), validator);
        vm.stopPrank();

        vm.deal(user, stakeAmount);
        vm.prank(user);

        // Check for the correct event
        vm.expectEmit(true, true, false, true);
        emit StakeReceived(address(stakingManager), user, stakeAmount);

        stakingManager.stake{value: stakeAmount}();

        assertEq(stakingManager.totalStaked(), stakeAmount);
        assertEq(kHYPE.balanceOf(user), stakeAmount);

        vm.prank(manager);
        stakingManager.enableWhitelist();

        // Approve and queue withdrawal
        vm.startPrank(user);
        kHYPE.approve(address(stakingManager), stakeAmount);
        // Store the withdrawalId (should be 0 for first withdrawal)
        uint256 withdrawalId = 0;
        stakingManager.queueWithdrawal(stakeAmount);
        vm.stopPrank();

        // Add ETH to the StakingManager contract to process the withdrawal
        vm.deal(address(stakingManager), stakeAmount);

        // Set withdrawal delay to 0 to allow immediate confirmation
        vm.prank(manager);
        stakingManager.setWithdrawalDelay(0);

        // Get the actual withdrawal request details to use in our event expectation
        StakingManager.WithdrawalRequest memory request = stakingManager.withdrawalRequests(user, withdrawalId);
        uint256 hypeAmount = request.hypeAmount;

        // Confirm withdrawal - use the actual hypeAmount from the request
        vm.prank(user);
        vm.expectEmit(true, true, true, true);
        emit WithdrawalConfirmed(user, withdrawalId, hypeAmount);
        stakingManager.confirmWithdrawal(withdrawalId);

        // Calculate the expected fee (0.1% of stakeAmount)
        uint256 feeRate = stakingManager.unstakeFeeRate();
        uint256 expectedFee = (stakeAmount * feeRate) / stakingManager.BASIS_POINTS();
        uint256 expectedClaimedAmount = stakeAmount - expectedFee;

        assertEq(stakingManager.totalQueuedWithdrawals(), 0);
        assertEq(stakingManager.totalClaimed(), expectedClaimedAmount, "Total claimed should account for fee");
    }
```

## Recommended mitigation:
To resolve this issue and ensure fair treatment of all users, the whitelist enforcement should be decoupled from withdrawals for existing stakers. Once a user has staked, their right to withdraw should be guaranteed.

A straightforward solution is to record a flag or timestamp for each user at the time they stake, indicating whether they were allowed to enter without a whitelist check.

Alternatively, the system could be refactored to support whitelist enforcement only at the point of deposit, not withdrawal. Once users have entered, they should not be blocked from exiting, provided they comply with all normal locking and timing rules.


## [M-1]Attacker gains rewards unfairly by a front run of rewards added into the system

## Finding description:

The reward distribution mechanism in the `StakingManager` contract introduces a vulnerability that can be exploited by attackers to gain a disproportionate share of staking rewards. This issue stems from the way the exchange ratio between `HYPE` and `kHYPE` tokens is calculated and used during staking and withdrawal operations.

At its core, the exchange ratio is computed dynamically based on the `total supply` of `kHYPE` tokens and the total amount of `HYPE` within the system. The exchange ratio is defined as follows:

```solidity
return Math.mulDiv(totalHYPE, 1e18, totalKHYPESupply);
```

This approach means that the exchange ratio increases immediately when rewards are added to the protocol. Since staking operations convert deposited HYPE into kHYPE using this exchange ratio, a higher exchange ratio results in fewer kHYPE tokens minted per unit of HYPE staked. Conversely, a lower exchange ratio results in more kHYPE being minted.

## Impact:
The core issue arises when a user—especially a **MEV bot—stakes a huge amount of HYPE** just before the `ValidatorManager` injects new rewards into the system. At this point, the exchange ratio is still low. As a result, the user receives more kHYPE tokens than they would if the reward had already been distributed. Once the reward is deposited, the total HYPE in the system increases, and so does the value of each kHYPE token. If the attacker now withdraws their position (or queues a withdrawal), they receive more HYPE than they initially staked, effectively capturing an unfair portion of the newly added rewards.

This exploit results in short-term attackers diluting the returns of long-term and honest stakers, reducing the protocol's fairness. Furthermore, since the system does not factor in how long someone has been staking—relying solely on snapshot values of total supply and rewards—there is no built-in defense against this timing-based exploitation.

## PoC:
Add this function to `StakingManager.t.sol` and run it with this command: `forge test --mt test_ExchangeRatio_WithStakeAndRewards -vvv`. To visualize how the Attacker (user) can claim rewards only for the lock period after `queueWithdrawal` and how the total rewards affect the user's individual rewards.
```solidity
function test_ExchangeRatio_WithStakeAndRewards() public {

        uint256 amount = 1 ether;

        // Setup validator and delegation
        vm.startPrank(manager);
        validatorManager.activateValidator(validator);
        validatorManager.setDelegation(address(stakingManager), validator);
        vm.stopPrank();

        // Setup initial stake
        vm.deal(user, amount);
        vm.prank(user);
        stakingManager.stake{value: amount}();


        // Simulate reward injection
        address testValidator = makeAddr("testValidator");
        vm.startPrank(admin);
        validatorManager.grantRole(validatorManager.MANAGER_ROLE(), admin);
        validatorManager.activateValidator(testValidator);
        vm.stopPrank();

        // Simulate a reward event
        vm.prank(address(oracleManager));
        validatorManager.reportRewardEvent(testValidator, 1e16);

        uint256 initialKHypeBalance = kHYPE.balanceOf(user);

        vm.prank(user);
        kHYPE.approve(address(stakingManager), amount);

        // Queue withdrawal
        vm.prank(user);
        stakingManager.queueWithdrawal(amount);

        // Verify withdrawal request
        StakingManager.WithdrawalRequest memory request = stakingManager.withdrawalRequests(user, 0);

        uint256 finalKHypeBalance = kHYPE.balanceOf(user);
        
        console.log("request.kHYPEAmount", request.kHYPEAmount);
        console.log("request.hypeAmount", request.hypeAmount);
    }
```

Also you are going to have to add:
```solidity
address public mockStakingManager;

function setUp() public override {
        super.setUp();

        //code...
        mockStakingManager = makeAddr("mockStakingManager");
        //code...
    }
```

## Exploit Example:
To illustrate this, consider the following simplified numbers:
- Total kHYPE supply before the attack: 100,000
- Total HYPE in the system before rewards: 20,000
- Exchange ratio: 0.2

An attacker deposits 1,000 HYPE just before validator rewards are sent to the contract. At this time, the attacker is minted:
- 1,000 / 0.2 = 5,000 kHYPE

Immediately after the stake, the protocol receives a validator reward of 5,000 HYPE. Now the total HYPE is 25,000, but the total kHYPE is 105,000, making the new exchange ratio:
- 25,000 / 105,000 ≈ 0.238

Now, if the attacker withdraws their 5,000 kHYPE using the updated exchange rate, they will receive:
- 5,000 * 0.238 ≈ 1,190 HYPE

This results in a net profit of 190 HYPE, which was not earned through legitimate staking, but rather by timing the reward inflow and abusing the dynamic ratio mechanism.

This scenario is effective even if the protocol has more fund into it but it is especially effective in the early days of deployment.

The impact is amplified by the fact that the system does not track how long funds have been staked. Users who stake for a single block and those who stake for weeks or months are treated the same in reward distribution. This design choice unintentionally encourages manipulative staking behavior and undermines the goal of promoting long-term participation.

## Recommended mitigation:
To address this vulnerability, the protocol should consider a fundamental redesign of how rewards are distributed and how staking entitlements are calculated.

1. One effective mitigation strategy is to adopt an epoch-based or time-weighted reward system. In such a system, rewards accrued during a specific epoch are only distributed to users who were staking before that epoch began. New stakers are only eligible for rewards starting from the next epoch. This approach ensures that users cannot benefit from staking right before a reward is issued, thereby removing the incentive for front-running.

2. Additionally, introducing a cooldown period before new stakers are eligible to withdraw or earn full rewards can deter short-term stake-and-withdraw strategies. Alternatively, the protocol can shift toward a time-weighted reward model, where each user’s reward is proportional not only to the amount staked but also to how long it has been staked.