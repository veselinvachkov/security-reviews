## Unreachable `depositRewards` Function

## Summary

The `depositRewards` function within the `StreamHandler` library is designed to distribute rewards over a specified dripping period. However, due to a logical flaw in the `isDrippingPeriodFinished` check, the function can never be successfully executed, rendering it effectively unusable. Initially, `lastDepositTimestamp` is implicitly set to 0 when the `RewardsStream` struct is created. The only way to set `lastDepositTimestamp` to create a depossit which is not possible because of a check in which `lastDepositTimestamp` takes part in.

## Finding Description

The `depositRewards` function begins with several checks, including `isDrippingPeriodFinished`:

Link to the affected code: https://github.com/sherlock-audit/2025-03-pinlink-rwa-tokenized-depin-marketplace-veselinvachkov/blob/754b25621f16420fc4a1cdae6a09dd055dcbcbf2/marketplace-contracts/src/marketplaces/streams.sol#L91
```solidity
function depositRewards(RewardsStream storage self, uint256 amount, uint256 drippingPeriod) internal {
    if (drippingPeriod > MAX_DRIPPING_PERIOD) revert PinlinkRewards_DrippingPeriodTooLong();
    if (drippingPeriod < MIN_DRIPPING_PERIOD) revert PinlinkRewards_DrippingPeriodTooShort();

    if (!self.isDrippingPeriodFinished()) revert PinlinkRewards_DepositRewardsTooEarly();
    if (!self.isEnabled()) revert PinlinkRewards_AssetNotEnabled();

    // ... rest of the function ...
}

function isDrippingPeriodFinished(RewardsStream storage self) internal view returns (bool) {
    return block.timestamp > self.lastDepositTimestamp + self.drippingPeriod;
}
```
The `isDrippingPeriodFinished` function compares the current `block.timestamp` to the sum of `lastDepositTimestamp` and `drippingPeriod`. Initially, `lastDepositTimestamp` is implicitly set to 0 when the `RewardsStream` struct is created.

The issue arises during the first call to `depositRewards`. To pass the `isDrippingPeriodFinished` check, the following condition must be true:

```solidity
block.timestamp > 0 + drippingPeriod
```
This condition can only be met if the `current block.timestamp` is greater than the specified `drippingPeriod`. However, `block.timestamp` can never be less than `drippingPeriod` alone.

Therefore, the `depositRewards` function will always revert with `PinlinkRewards_DepositRewardsTooEarly` on its first invocation. Also this breaks the implementation of the function inside `PinlinkShop`.

## Impact Explanation
The impact is High. The intended functionality of distributing rewards over time is completely broken. The `depositRewards` function, a core component for reward distribution, is unusable. 
https://github.com/sherlock-audit/2025-03-pinlink-rwa-tokenized-depin-marketplace-veselinvachkov/blob/754b25621f16420fc4a1cdae6a09dd055dcbcbf2/marketplace-contracts/src/marketplaces/pinlinkShop.sol#L214

## Likelihood Explanation
The likelihood is Certain. The logical flaw is inherent in the design and will always prevent the `depositRewards` function from being called successfully.

## Proof of Concept
Deploy a contract that uses the `StreamHandler` library.
Enable an asset using `enableAsset`.
Attempt to call `depositRewards` with any valid parameters.
The transaction will always revert with `PinlinkRewards_DepositRewardsTooEarly`.

## Recommendation
Modify the `isDrippingPeriodFinished` check to handle the initial state correctly. One approach is to introduce a flag indicating whether the first deposit has been made. By introducing `firstDepositMade`, the first call to `depositRewards` will pass the `isDrippingPeriodFinished` check, and subsequent calls will follow the intended dripping period logic because now there is going to be a set `self.lastDepositTimestamp`.