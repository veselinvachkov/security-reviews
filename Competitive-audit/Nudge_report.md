**Title: Nudge.xyz**

Auditor: **vesko210**

## Issues found
|Severtity|Number of issues found
| ------- | -------------------- |
| Medium  | 2                    |

# Findings

## [M-1]Treasury update results in permanent fund loss

### Description
The `updateTreasuryAddress` function allows the administrator to change the treasury address to a new one. However, when this update occurs, any accumulated funds (both ERC20 tokens and native Ether) in the old treasury are not transferred to the new treasury. As a result, any funds held by the previous treasury will become inaccessible, effectively leading to their loss.

### Impact
This issue can lead to a complete loss of treasury funds if the update function is called without manually transferring funds beforehand. Any fees or rewards collected in the old treasury will no longer be usable by the protocol. If this function is executed without additional safeguards, the protocol’s financial operations could be severely disrupted, leading to irreversible loss of capital.

### Proof of Concept

The problematic function is found in the contract:
```solidity
/// @notice Updates the treasury address
/// @param newTreasury New address for the treasury
/// @dev Only callable by NUDGE_ADMIN_ROLE
function updateTreasuryAddress(address newTreasury) external onlyRole(NUDGE_ADMIN_ROLE) {
    if (newTreasury == address(0)) revert InvalidTreasuryAddress();

    //@audit accumulated funds are not transferred to the new treasury
    address oldTreasury = nudgeTreasuryAddress;
    nudgeTreasuryAddress = newTreasury;

    emit TreasuryUpdated(oldTreasury, newTreasury);
}
```

The function only updates the treasury address but does not transfer any assets from the old treasury to the new one.

The following test case shows that after updating the treasury address, funds remain locked in the old treasury and are not automatically transferred.
- You can add this test to `NudgeCampaignFactoryTest` test file:
```solidity
function test_UpdateTreasuryAddressWillLoseAllFunds() public {
    address newTreasury = address(0x999);

    // Deploy and fund 2 campaigns
    uint256 campaignOneRewards = 100 ether;
    uint256 campaignTwoRewards = 200 ether;

    rewardToken.mintTo(campaignTwoRewards, address(this));
    rewardToken.approve(address(factory), campaignTwoRewards);

    // Campaign 1: NATIVE TOKEN for rewards
    address payable campaignOne = payable(
        factory.deployAndFundCampaign{value: campaignOneRewards}(
            holdingPeriod,
            address(targetToken),
            factory.NATIVE_TOKEN(),
            REWARD_PPQ,
            campaignAdmin,
            0,
            withdrawalAddress,
            campaignOneRewards,
            1
        )
    );

    // Campaign 2: ERC20 Token for rewards
    address payable campaignTwo = payable(
        factory.deployAndFundCampaign(
            holdingPeriod,
            address(targetToken),
            address(rewardToken),
            REWARD_PPQ,
            campaignAdmin,
            0,
            withdrawalAddress,
            campaignTwoRewards,
            2
        )
    );

    // Generate fees
    uint256 toAmount = 100e18;

    simulateReallocation(campaignOne, toAmount, 1);
    simulateReallocation(campaignTwo, toAmount, 2);

    // Collect fees
    address ;
    campaigns[0] = campaignOne;
    campaigns[1] = campaignTwo;

    vm.prank(operator);
    factory.collectFeesFromCampaigns(campaigns);

    uint256 treasuryBalanceBeforeInRewardToken = rewardToken.balanceOf(treasury);
    uint256 treasuryBalanceBeforeInEther = address(treasury).balance;

    // Should succeed with valid parameters
    vm.prank(admin);
    factory.updateTreasuryAddress(newTreasury);
    assertEq(factory.nudgeTreasuryAddress(), newTreasury);

    // Assert that the new treasury received the funds from the old treasury
    assertEq(rewardToken.balanceOf(newTreasury), treasuryBalanceBeforeInRewardToken, "New treasury should receive all RewardTokens");
    assertEq(address(newTreasury).balance, treasuryBalanceBeforeInEther, "New treasury should receive all Ether");

    // Assert that the old treasury no longer holds any funds
    assertEq(rewardToken.balanceOf(treasury), 0, "Old treasury should have zero RewardTokens");
    assertEq(address(treasury).balance, 0, "Old treasury should have zero Ether");
}
```

- The test case demonstrates that funds remain in the old treasury unless manually transferred before updating the address.

### Severity:
The Likelihood is low as there might be a manuall way of transfering the funds, but the impact is high. When changeing the treasury address you always need to ensure there are no funds left inside of the old address.

### Recommended Mitigation Steps

**Automatic Fund Transfer:** Modify the `updateTreasuryAddress` function to transfer all funds from the old treasury to the new treasury before updating the address.

**Add Precondition Checks:** Introduce a check that prevents updating the treasury address if it still holds funds.

## [M-2]Fee evasion is possible via small reallocations

### Finding description

In the `NudgeCampaign` contract where users can bypass fees by making small reallocations. This issue arises due to the interaction between the `getRewardAmountIncludingFees` and `calculateUserRewardsAndFees` functions when processing amounts below a certain threshold.

The problem occurs when `rewardPPQ` is set to 2e13 (as it is during test scenarios). During reallocation:
```solidity
        function handleReallocation(...) external payable whenNotPaused {

        //code...

@>      if (amountReceived < toAmount) {
            revert InsufficientAmountReceived();
        }

        _transfer(toToken, userAddress, amountReceived);

        totalReallocatedAmount += amountReceived;

@>      uint256 rewardAmountIncludingFees = getRewardAmountIncludingFees(amountReceived);

        uint256 rewardsAvailable = claimableRewardAmount();
        if (rewardAmountIncludingFees > rewardsAvailable) {
            revert NotEnoughRewardsAvailable();
        }

@>      (uint256 userRewards, uint256 fees) = calculateUserRewardsAndFees(rewardAmountIncludingFees);
        pendingRewards += userRewards;
        accumulatedFees += fees;

        //code...
```
1. If `amountReceived` < 500 tokens, `getRewardAmountIncludingFees` gets called with `amountReceived` < 500 calculates the reward based on `amountReceived` and returns 2% of it.
```solidity
//        499     *     2e13     /    1e15 = 9
return toAmount.mulDiv(rewardPPQ, PPQ_DENOMINATOR);
```
2. This value is then used in `calculateUserRewardsAndFees`, where `rewardAmountIncludingFees` < 10.
```solidity
    function calculateUserRewardsAndFees(
        uint256 rewardAmountIncludingFees
    ) public view returns (uint256 userRewards, uint256 fees) {
        //  fees =                 9     *   1_000   /  10_000 = 0 
        fees = (rewardAmountIncludingFees * feeBps) / BPS_DENOMINATOR;
        userRewards = rewardAmountIncludingFees - fees;
    }
```
3. Given the default feeBps = 1000 and BPS_DENOMINATOR = 10000, the fee calculation results in zero fees being charged.

4. Consequently, users receive the full reward amount without incurring any fees.

This loophole allows users to systematically evade fees by making repeated small reallocations(which in this basic case is not that small, 500 tokens is not a small amont depending on their price), leading to revenue losses for the protocol. Also, the threshold for skipping the fee varies depending on the ratio between "reward" and "fees".

### Impact
- Users can evade fees by splitting transactions into smaller amounts.

- The protocol loses revenue from fees that would otherwise be charged on reallocations.

### Recommended mitigation steps

Introduce a minimum fee that applies regardless of the transferred amount to ensure every transaction incurs some cost.

Example: `feeAmount = max((rewardAmountIncludingFees * feeBps) / BPS_DENOMINATOR, MINIMUM_FEE);`
