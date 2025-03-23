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

### Proof of Concept

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

The `withdrawRewards` function allows the `CAMPAIGN_ADMIN_ROLE` to withdraw any available rewards. The constructor allows any user to be assigned the `CAMPAIGN_ADMIN_ROLE`. This means that the campaign creator can arbitrarily withdraw all available rewards at any time, potentially depleting the reward pool before any participants can participate. This can lead to a scenario where users can not participate in the campaign (after `withdrawRewards` is called), effectively breaking the campaign's intended functionality and damaging user trust. Also there can be unlimited number of campaigns and if most of them are griefed with the `withdrawRewards` users will not be able to join a honorable campaign which leads to DOS.

The contract first has to whitelist which campaigns are available on nudge.xyz but thay can't know if `CAMPAIGN_ADMIN` will troll the campaign, and given the fact that someone can create an automated algorithm that deploys campaigns, the nudge.xyz team will be stuck with an unlimited number of troll campaigns to choose from, and the eligible one could be a grain of sand in the sea.

The gas estimate for deploying this contract is going to be between 1,500,000 to 2,500,000 gas, depending on the Ethereum network conditions at the time of deployment. This means that for a attacker to deploy 100_000_000_000 campaigns (each with 2,000,000 gas, which can be lowered a lot given the fact that the attacker is going to create the attack in his favour) he will have to pay around 390-400 usd for the current price of ETH. That many contract are going to cause absolutely significant problems for the nudge.xyz team to remove all attack requests.

Also, if the nudge.xyz team had a way to delete all deployments waiting for approval, it would harm honest users of the protocol, and that doesn't stop the attacker from deploying (relatively) small batches of 100_000_000 campaigns to handle this case.
```solidity 
    /// @notice Deploys a new NudgeCampaign contract
    /// @param holdingPeriodInSeconds Duration users must hold tokens to be eligible for rewards
    /// @param targetToken Address of the token users need to hold
    /// @param rewardToken Address of the token used for rewards
    /// @param rewardPPQ The reward factor in parts per quadrillion for calculating rewards
    /// @param campaignAdmin Address of the campaign admin
    /// @param startTimestamp When the campaign starts
    /// @param alternativeWithdrawalAddress Optional address for alternative reward withdrawal
    /// @param uuid Unique identifier for the campaign
    /// @return campaign Address of the deployed campaign contract
    /// @dev Uses Create2 for deterministic address generation
    function deployCampaign(
        uint32 holdingPeriodInSeconds,
        address targetToken,
        address rewardToken,
        uint256 rewardPPQ,
        address campaignAdmin,
        uint256 startTimestamp,
        address alternativeWithdrawalAddress,
        uint256 uuid
    ) public returns (address campaign) {
        if (campaignAdmin == address(0)) revert ZeroAddress();
        if (targetToken == address(0) || rewardToken == address(0)) revert ZeroAddress();
        if (holdingPeriodInSeconds == 0) revert InvalidParameter();

        // Generate deterministic salt using all parameters
        bytes32 salt = keccak256(
            abi.encode(
                holdingPeriodInSeconds,
                targetToken,
                rewardToken,
                rewardPPQ,
                campaignAdmin,
                startTimestamp,
                FEE_BPS,
                alternativeWithdrawalAddress,
                uuid
            )
        );

        // Create constructor arguments
        bytes memory constructorArgs = abi.encode(
            holdingPeriodInSeconds,
            targetToken,
            rewardToken,
            rewardPPQ,
            campaignAdmin,
            startTimestamp,
            FEE_BPS,
            alternativeWithdrawalAddress,
            uuid
        );

        // Deploy using CREATE2
        bytes memory bytecode = abi.encodePacked(type(NudgeCampaign).creationCode, constructorArgs);
        campaign = Create2.deploy(0, salt, bytecode);

        // Track the campaign
        isCampaign[campaign] = true;
        campaignAddresses.push(campaign);

        emit CampaignDeployed(campaign, campaignAdmin, targetToken, rewardToken, startTimestamp, uuid);
    }
```
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
- **Reputational damage and DOS:** The nudge.xyz team faces a severe denial-of-service (DoS) threat, as attackers can flood the platform with an unlimited number of malicious campaigns, making it nearly impossible to identify and approve legitimate ones. With the low cost of execution, the team would be forced into an endless battle of filtering out fraudulent deployments, crippling their ability to maintain a functional and trustworthy platform. As a result, users would struggle to find, participate and most effectively create in genuine campaigns, ultimately losing trust in the ecosystem.

### Recommendation:
- Add a small fee for campaign deployment, the protocol is going to earn small revenue and the griefing attack is never going to be possible. The fee can be as samll as 0.10 usd, this way the attacker will have to spend 10_000 usd to deploy just 100_000 campaigns which is a huge decrease from the example i gave above in the description" section.
- Create a new role that will be trusted to create campaigns, this will drastically reduce the possibility of greifing for attackers.
- Deactivate the campaign every time `withdrawRewards` is called this way, you will free up space for new campaigns.

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
