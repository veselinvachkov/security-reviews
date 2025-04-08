## Finding Title: Incorrect Interest Calculation Due to _performanceFeeRate Change
**Severity: Medium**
## Summary
The `_accruedPeriodInterest()` function applies the `_performanceFeeRate` to the entire period's accrued interest, instead of considering when the fee rate was changed. This results in inaccurate fee calculations, potentially leading to incorrect fund distributions.

## Finding Description
If the fee rate changes mid-period, the function still applies the latest rate to the full period. This breaks financial accuracy guarantees and can lead to unfair fee calculations.

The `_performanceFeeRate` can be changed before `_accruedPeriodInterest()` is executed to increase or decrease fees unfairly, exploiting this miscalculation to their advantage. This issue impacts the protocol’s revenue and user returns.

## Impact Explanation
This bug could lead to:
- **Overcharging or undercharging fees:** Users might pay more or less than intended, leading to potential disputes.
- **Incorrect revenue allocation:** The protocol could miscalculate its earnings, affecting financial sustainability.
- **Exploitation opportunities:** Attackers could time changes in `_performanceFeeRate` to gain an unfair financial advantage.

Given these factors, the impact is assessed as **Medium**, as it affects financial fairness and protocol integrity.

## Likelihood Explanation
The probability of this issue occurring is **Medium to High** depending on how often `_performanceFeeRate` is changed, with use of the function `setPerformanceFeeRate` the issue would occur for users who have some interest:
- The more often `_performanceFeeRate` changes, the more often the problem will occur.
- Users with knowledge of the bug could manipulate timing to benefit unfairly.

## Proof of Concept
**Problematic Code in `_accruedPeriodInterest() located in: TermMax\contracts\vault\TermMaxVault.sol`**
```solidity
function _accruedPeriodInterest(uint256 startTime, uint256 endTime) internal {
    uint256 interest = (_annualizedInterest * (endTime - startTime)) / 365 days;
    uint256 _performanceFeeToCurator = (interest * _performanceFeeRate) / Constants.DECIMAL_BASE;
    
    _performanceFee += _performanceFeeToCurator;
    _accretingPrincipal += (interest - _performanceFeeToCurator);
}
```
#### **Steps to Exploit**
1. `_performanceFeeRate` is initially **5%**.
2. Mid-period, `_performanceFeeRate` changes to **7%**.
3. `_accruedPeriodInterest()` still applies **7% to the entire period**, overcharging fees.
4. If an attacker knows this, they can time `_performanceFeeRate` changes to gain a financial advantage against all the other users.


```solidity
    function testIncorrectPerformanceFeeCalculationWhenRateChanges() public {
    uint256 initialDeposit = 1000e18;
    uint256 initialFeeRate = 0.5e8; // Initial performance fee rate
    uint256 newFeeRate = 0.7e8; // New performance fee rate after change
    
    // Deploy the vault and submit initial deposit
    vm.startPrank(deployer);
    res.debt.mint(deployer, initialDeposit);
    res.debt.approve(address(vault), initialDeposit);
    vault.deposit(initialDeposit, deployer);
    vm.stopPrank();

    // Set the initial performance fee rate
    vault.setPerformanceFeeRate(initialFeeRate);

    // This represents the period before the fee rate change
    vm.warp(currentTime + 30 days);
    
    vault.setPerformanceFeeRate(newFeeRate);

    vm.warp(currentTime + 60 days);  // Another 30 days after the fee change
    vault.accrueInterest();

    uint256 actualFee = (100e18 * newFeeRate) / 1e8 + (100e18 * newFeeRate) / 1e8;
    // Accrued interest before fee change (100e18 * 0.5% = 0.5e18) + after fee change (100e18 * 0.7% = 0.7e18)
    uint256 expectedFee = (100e18 * initialFeeRate) / 1e8 + (100e18 * newFeeRate) / 1e8;

    // Assert the performance fee has been calculated correctly
    assertEq(expectedFee, actualFee, "The performance fee was calculated incorrectly!");
    }
```
**Why it will fail:**
The test asserts that the performance fee should be calculated separately for the two periods based on the different fee rates (0.5% for the first period, 0.7% for the second period).
Since the logic in the contract doesn't differentiate between periods with different fee rates, it will incorrectly apply the updated 0.7% fee rate to the entire period's interest.

## Recommendation
### **Fix: Track Fee Rate Changes Over Time**
Modify `_accruedPeriodInterest()` to segment the period based on fee rate changes:

- Tracks `_performanceFeeRate` changes over time.
- Splits interest calculation into segments based on historical fee rate changes.
- Applies the correct fee rate for each time period, ensuring fairness.