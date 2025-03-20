**Title: Nudge.xyz**

Auditor: **vesko210**

## Issues found
|Severtity|Number of issues found
| ------- | -------------------- |
| High    | 1                    |
| Medium  | 3                    |

# Findings

## [H-1]Treasury update results in permanent fund loss

### Description
The `updateTreasuryAddress` function allows the administrator to change the treasury address to a new one. However, when this update occurs, any accumulated funds (both ERC20 tokens and native Ether) in the old treasury are not transferred to the new treasury. As a result, any funds held by the previous treasury will become inaccessible, effectively leading to their loss.

### Impact
This issue can lead to a complete loss of treasury funds if the update function is called without manually transferring funds beforehand. Any fees or rewards collected in the old treasury will no longer be usable by the protocol. If this function is executed without additional safeguards, the protocol’s financial operations could be severely disrupted, leading to irreversible loss of capital.

## Proof of Concept

### Code Reference
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

### Test Case Demonstrating the Issue
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

## Recommended Mitigation Steps

**Automatic Fund Transfer:** Modify the `updateTreasuryAddress` function to transfer all funds from the old treasury to the new treasury before updating the address.

**Add Precondition Checks:** Introduce a check that prevents updating the treasury address if it still holds funds.


## [M-1]Unrestricted campaign creation and reward withdrawal by campaign admin leads to DOS

### Finding description

The `withdrawRewards` function allows the `CAMPAIGN_ADMIN_ROLE` to withdraw any available rewards. The constructor allows any user to be assigned the `CAMPAIGN_ADMIN_ROLE`. This means that the campaign creator can arbitrarily withdraw all available rewards at any time, potentially depleting the reward pool before other participants can participate and claim rewards. This can lead to a scenario where users participate in the campaign (after `withdrawRewards` is called) but receive no rewards, effectively breaking the campaign's intended functionality and damaging user trust. Also there can be unlimited number of campaigns and if most of them are griefed with the `withdrawRewards` users will not be able to join a honorable campaign which leads to DOS.

```solidity
    //@audit CAMPAIGN_ADMIN_ROLE can be a normal user that created the campaign
    function withdrawRewards(uint256 amount) external onlyRole(CAMPAIGN_ADMIN_ROLE) {
        if (amount > claimableRewardAmount()) {
            revert NotEnoughRewardsAvailable();
        }

        address to = alternativeWithdrawalAddress == address(0) ? msg.sender : alternativeWithdrawalAddress;

        _transfer(rewardToken, to, amount);

        emit RewardsWithdrawn(to, amount);
    }
```
After the function above is called with the full amount returned by `claimableRewardAmount()` this one will always return 0 value: 
```solidity
    function claimableRewardAmount() public view returns (uint256) {
        return getBalanceOfSelf(rewardToken) - pendingRewards - accumulatedFees;
    }
```
### Impact:

- **Lack of rewards for participants:** Users will not be able participate in the campaign after the call of `withdrawRewards` because the rewards pool will have only funds that are distributed for `pendingRewards` and for `accumulatedFees`. This creates ghost campaigns.

- **Reputational damage:** The project's reputation can be damaged if users abuse `withdrawRewards` which will lead to many campaigns in which rewards cannot be earned. There is no limit to the possible number of campaigns, which can also result in users not being able to find a campaign that offers them rewards because there are simply too many griefed campaigns and thus users will perceive the campaigns as dishonest or manipulative. (Users will not be able to find legitimate campaigns and will not be able to participate in any because of this.)

The only way to deactivate campaigns is when `NUDGE_ADMIN_ROLE` calls `setIsCampaignActive`, but that is a slow and manuall way, which can not catch up with a automated campaign deployment ment for griefing.
### Recommendation:

**Restrict `withdrawRewards` to Campaign End:**

- Implement a `campaignEndTimestamp` in the contract.
- Modify the `withdrawRewards` function to only allow withdrawals after `block.timestamp >= campaignEndTimestamp`. This ensures rewards are only withdrawn after the campaign concludes and there can be no griefing from the `CAMPAIGN_ADMIN_ROLE` and also this can solve the problem with the unlimited campaigns that can be created as the old ones will be not active.

- Also another idea is to deactivate the campaign every time `withdrawRewards` gets called.

## [M-2]Lack of validation on `targetToken` and `rewardToken` allows deployment with unsupported ERC20 tokens

### Description
The `NudgeCampaign` contract does not enforce validation on the `targetToken` and `rewardToken` parameters in the constructor. This means that the deployer can specify non-standard ERC20 tokens (e.g., fee-on-transfer or rebasing tokens) as the `targetToken` or `rewardToken`. Such tokens are explicitly unsupported by the protocol, as stated in the documentation:

> "Nudge does not support non-standard ERC20 tokens (fee-on-transfer, rebasing tokens…) and ERC20 tokens with a value returned by `decimals()` that is inaccurate or strictly above 18."

Since no verification is performed in the constructor, campaigns can be created with these unsupported tokens, potentially leading to incorrect reward calculations, misallocated funds, and financial losses for participants.

```solidity
constructor(
        uint32 holdingPeriodInSeconds_,
        address targetToken_,
        address rewardToken_,
        uint256 rewardPPQ_,
        address campaignAdmin,
        uint256 startTimestamp_,
        uint16 feeBps_,
        address alternativeWithdrawalAddress_,
        uint256 campaignId_
    ) {
        if (rewardToken_ == address(0) || campaignAdmin == address(0)) {
            revert InvalidCampaignSettings();
        }

        //code...

        //@audit targetToken_ and rewardToken_ are not validated if they are supported by the protocol
@>      targetToken = targetToken_;
@>      rewardToken = rewardToken_;
        campaignId = campaignId_;

        // Compute scaling factors based on token decimals
        uint256 targetDecimals = targetToken_ == NATIVE_TOKEN ? 18 : IERC20Metadata(targetToken_).decimals();
        uint256 rewardDecimals = rewardToken_ == NATIVE_TOKEN ? 18 : IERC20Metadata(rewardToken_).decimals();

        // Calculate scaling factors to normalize to 18 decimals
        targetScalingFactor = 10 ** (18 - targetDecimals);
        rewardScalingFactor = 10 ** (18 - rewardDecimals);

        //code...
    }
```

### Impact
- **Incorrect reward calculations:** If a fee-on-transfer token is used, the contract may assume an incorrect balance for reward calculations, leading to either excess or insufficient rewards.
- **Rebasing token inconsistencies:** Rebasing tokens dynamically change supply, which can cause participation tracking to become unreliable.
- **Potential loss of funds:** If the contract assumes an incorrect token balance, rewards may be miscalculated, leading to underpayment or overpayment of users.
- **System instability:** Non-standard tokens may introduce unexpected behaviors that disrupt the campaign’s operation.

### Proof of Concept

**Step 1: Deploying a Campaign with a Fee-on-Transfer Token:** A malicious deployer can set a fee-on-transfer token as the `targetToken` or `rewardToken`.

**Step 2: Deploying `NudgeCampaign` with the Malicious Token:** A campaign deployer initializes `NudgeCampaign` with this fee-on-transfer token.

**Step 3: Users And Contract Suffer Unexpected Fees:**
- Users who attempt to participate will send a certain amount of tokens.
- However, they receive less than expected due to the transfer fee.
- The contract assumes the full amount was received, leading to incorrect reward calculations.

### Recommended Mitigation
To prevent non-standard ERC20 tokens from being set as `targetToken` or `rewardToken`, the constructor should include checks that stop the deployment if non-standard ERC20 tokens are used.

## [M-3]Fee evasion is possible via small reallocations

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
