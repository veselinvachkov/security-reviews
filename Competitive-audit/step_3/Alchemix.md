**Title: Alchemix**

Auditor: **vesko210**

## Issues found
|Severtity|Number of issues found
| ------- | -------------------- |
| High    | 2                    |


# Findings

# [H-1] Earmarked debt repayments during liquidation fail to fund transmuter when incorrect recipient is used


## Summary

Earmarked debt repayments during liquidation fail to transfer yield tokens to the transmuter when an incorrect recipient is specified, leading to a state where internal accounting reflects repayment, but the transmuter does not receive the corresponding funds.

---

## Finding Description

In the `_forceRepay` or `liquidate()` logic of the Alchemist system, when a user liquidates a position with earmarked debt, the internal state correctly reduces the `collateralBalance` and `earmarked` values of the account. However, if the yield tokens used to repay this earmarked debt are not explicitly transferred to the transmuter contract, they are effectively lost to the system — breaking the invariant that all earmarked debt is backed by yield-bearing assets.

When the liquidation flow uses `address(this)` as the recipient of the repaid tokens instead of `transmuter`, the system updates internal accounting but does not actually transfer the repaid yield tokens to the transmuter. This discrepancy leads to a mismatch where the transmuter believes it is owed tokens that it never received.

---

## Impact Explanation

The issue allows a liquidation to "burn" earmarked debt without actually funding the transmuter with the corresponding collateral. This leads to insolvency over time as the transmuter becomes underfunded while its internal state assumes funds are present.

**Impact: High**

---

## Likelihood Explanation

The issue occurs every time a position gets liquidated and `_forceRepay` gets called:

```solidity
if (account.earmarked > 0) {
    repaidAmountInYield = _forceRepay(accountId, convertDebtTokensToYield(account.earmarked));
}
````

**Likelihood: High**

---

# Proof of Concept

Add this function to the `AlchemistV3.t.sol` test file:
`testLiquidate_Undercollateralized_Position_With_Earmarked_Debt_Liquidation_Logs`

```solidity
function testLiquidate_Undercollateralized_Position_With_Earmarked_Debt_Liquidation_Logs() external {
    uint256 amount = 200_000e18; // 200,000 yvdai
    vm.startPrank(someWhale);
    fakeYieldToken.mint(whaleSupply, someWhale);
    vm.stopPrank();

    vm.startPrank(yetAnotherExternalUser);
    SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), amount * 2);
    alchemist.deposit(amount, yetAnotherExternalUser, 0);
    vm.stopPrank();

    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), amount + 100e18);
    alchemist.deposit(amount, address(0xbeef), 0);
    uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    uint256 mintAmount = alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization;
    alchemist.mint(tokenIdFor0xBeef, mintAmount, address(0xbeef));
    vm.stopPrank();

    vm.startPrank(anotherExternalUser);
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount);
    transmuterLogic.createRedemption(mintAmount);
    vm.stopPrank();

    vm.roll(block.number + (5_256_000 * 5 / 100));

    (, uint256 prevDebt, uint256 earmarked) = alchemist.getCDP(tokenIdFor0xBeef);
    require(earmarked == prevDebt * 5 / 100, "Earmarked debt should be 60% of the total debt");

    uint256 initialVaultSupply = IERC20(address(fakeYieldToken)).totalSupply();
    fakeYieldToken.updateMockTokenSupply(initialVaultSupply);
    uint256 modifiedVaultSupply = (initialVaultSupply * 590 / 10_000) + initialVaultSupply;
    fakeYieldToken.updateMockTokenSupply(modifiedVaultSupply);

    vm.assertApproxEqAbs(prevDebt, 180_000_000_000_000_000_018_000, minimumDepositOrWithdrawalLoss);

    uint256 alchemistBalanceYieldBefore = fakeYieldToken.balanceOf(address(alchemist));
    uint256 transmuterBalanceYieldBefore = fakeYieldToken.balanceOf(address(transmuterLogic));

    vm.startPrank(externalUser);
    uint256 liquidatorPrevTokenBalance = IERC20(fakeYieldToken).balanceOf(address(externalUser));
    uint256 liquidatorPrevUnderlyingBalance = IERC20(fakeUnderlyingToken).balanceOf(address(externalUser));

    uint256 collateralAfterRepayment = alchemist.totalValue(tokenIdFor0xBeef) - earmarked;
    uint256 collateralAfterRepaymentInYield = alchemist.convertUnderlyingTokensToYield(collateralAfterRepayment);
    uint256 debtAfterRepayment = prevDebt - earmarked;
    uint256 earmarkedInYield = alchemist.convertDebtTokensToYield(earmarked);
    uint256 alchemistCurrentCollateralization =
        alchemist.normalizeUnderlyingTokensToDebt(alchemist.getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / alchemist.totalDebt();

    (uint256 liquidationAmount, uint256 expectedDebtToBurn, uint256 expectedBaseFee) = alchemist.calculateLiquidation(
        collateralAfterRepayment,
        debtAfterRepayment,
        alchemist.minimumCollateralization(),
        alchemistCurrentCollateralization,
        alchemist.globalMinimumCollateralization(),
        liquidatorFeeBPS
    );

    uint256 expectedLiquidationAmountInYield = alchemist.convertDebtTokensToYield(liquidationAmount);
    uint256 expectedBaseFeeInYield = alchemist.convertDebtTokensToYield(expectedBaseFee);
    uint256 expectedFeeInUnderlying = expectedDebtToBurn * liquidatorFeeBPS / 10_000;

    (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenIdFor0xBeef);
    uint256 liquidatorPostTokenBalance = IERC20(fakeYieldToken).balanceOf(address(externalUser));
    uint256 liquidatorPostUnderlyingBalance = IERC20(fakeUnderlyingToken).balanceOf(address(externalUser));
    (uint256 depositedCollateral, uint256 debt,) = alchemist.getCDP(tokenIdFor0xBeef);
    uint256 alchemistBalanceYieldAfter = fakeYieldToken.balanceOf(address(alchemist));
    uint256 transmuterBalanceYieldAfter = fakeYieldToken.balanceOf(address(transmuterLogic));

    vm.stopPrank();

    console.log("alchemistBalanceYieldBefore: ", alchemistBalanceYieldBefore);
    console.log("transmuterBalanceYieldBefore: ", transmuterBalanceYieldBefore);
    console.log("alchemistBalanceYieldAfter: ", alchemistBalanceYieldAfter);
    console.log("transmuterBalanceYieldAfter: ", transmuterBalanceYieldAfter);
}
```

---

## How to Use the Test

1. Run the test with the recipient set to `address(this)` inside `_forceRepay`:

```solidity
TokenUtils.safeTransfer(yieldToken, address(this), creditToYield);
```

2. Then run it with the recipient set to the `transmuter`:

```solidity
TokenUtils.safeTransfer(yieldToken, transmuter, creditToYield);
```

---

## Result Log Comparison

**Correct recipient (`transmuter`):**

```
alchemistBalanceYieldAfter:     290985999999999998125634
transmuterBalanceYieldAfter:    108732600000000001880417
```

**Incorrect recipient (`address(this)`):**

```
alchemistBalanceYieldAfter:     300516999999999998135716
transmuterBalanceYieldAfter:     99201600000000001870335
```

Despite identical changes in internal accounting (`collateralBalance` and `earmarked`), the transmuter receives \~10M fewer yield tokens in the broken version — indicating the funds were never delivered.

---

## Recommendation

Ensure that yield tokens used to repay earmarked debt are always sent to the transmuter. In the liquidation logic, update the `_forceRepay` or equivalent call to include the correct recipient.

---

## Fix Snippet

```solidity
TokenUtils.safeTransfer(yieldToken, transmuter, creditToYield); // Correct
```


# [H-1] Incorrect Protocol Fee Value in `repay()` Results in Full `creditToYield` Being Sent to Fee Receiver

## Summary

The `repay()` function in the `AlchemistV3` contract miscalculates the protocol fee by sending the entire `creditToYield` amount to the `protocolFeeReceiver`, rather than the correct fee based on the configured `protocolFee`. This results in overpayment from the protocol and a loss of funds.

---

## Finding Description

In the current implementation of `AlchemistV3.repay()`, the function incorrectly performs the following transfers:

```solidity
// Incorrect logic: full amount sent as fee
TokenUtils.safeTransferFrom(yieldToken, msg.sender, transmuter, creditToYield);
TokenUtils.safeTransfer(yieldToken, protocolFeeReceiver, creditToYield);
````

Instead, it should calculate and transfer only the fee, similar to the logic in the `redeem()` function:

```solidity
uint256 fee = creditToYield * protocolFee / BPS;
TokenUtils.safeTransfer(yieldToken, protocolFeeReceiver, fee);
```

---

## Impact Explanation

**Impact: High**

The bug causes misallocation of funds on every call to `repay()`. The entire repayment amount is incorrectly sent as a fee, depleting the protocol's token reserves. Over time, this will likely lead to insolvency or an inability to fulfill user withdrawals.

---

## Likelihood Explanation

**Likelihood: High**

This issue occurs on every call to `repay()` — a core part of user interaction with the protocol. Since the logic is unconditional and affects all users, it's guaranteed to be triggered frequently.

---

## Proof of Concept

Given:

* A user deposits and mints a debt position.
* They repay the debt with `repay(100e18, tokenId)`, where `creditToYield = 50e18`.
* The protocol fee is set at 10% (`protocolFee = 1000`, `BPS = 10000`).

**Expected behavior:**

* `protocolFeeReceiver` should receive `50e18 * 10% = 5e18`.

**Actual behavior:**

* `protocolFeeReceiver` receives **50e18** instead of 5e18.

---

## Test Case

Add the following to `AlchemistV3.t.sol` and run with `forge test --mt testRepayUnearmarkedFee -vvv`:

```solidity
function testRepayUnearmarkedFee() external {
    uint256 amount = 100e18;

    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), amount + 100e18);
    alchemist.deposit(amount, address(0xbeef), 0);
    uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    alchemist.mint(tokenId, amount / 2, address(0xbeef));

    console.log("protocolFeeReceiver: ", fakeYieldToken.balanceOf(alchemist.protocolFeeReceiver()));
    console.log("transmuterLogic: ", fakeYieldToken.balanceOf(address(transmuterLogic)));
    console.log("alchemist: ", fakeYieldToken.balanceOf(address(alchemist)));

    uint256 preRepayBalance = fakeYieldToken.balanceOf(address(0xbeef));
    vm.roll(block.number + 1);

    uint256 amountSent = alchemist.repay(100e18, tokenId);
    vm.stopPrank();

    (, uint256 userDebt,) = alchemist.getCDP(tokenId);
    assertEq(userDebt, 0);

    assertEq(fakeYieldToken.balanceOf(address(transmuterLogic)), alchemist.convertDebtTokensToYield(amount / 2));
    assertEq(fakeYieldToken.balanceOf(address(0xbeef)), preRepayBalance - alchemist.convertDebtTokensToYield(amount / 2));
    assertEq(fakeYieldToken.balanceOf(alchemist.protocolFeeReceiver()), amountSent);

    console.log("protocolFeeReceiver: ", fakeYieldToken.balanceOf(alchemist.protocolFeeReceiver()));
    console.log("transmuterLogic: ", fakeYieldToken.balanceOf(address(transmuterLogic)));
    console.log("alchemist: ", fakeYieldToken.balanceOf(address(alchemist)));

    assert(address(transmuterLogic) != address(10));
}
```

---

## Actual Log Output

**Before Repay:**

```
protocolFeeReceiver:  0
transmuterLogic:      0
alchemist:            100000000000000000000
```

**After Repay:**

```
protocolFeeReceiver:  50000000000000000000 // 50e18 received
transmuterLogic:      50000000000000000000 // 50e18 received
alchemist:            50000000000000000000 // 50e18 reduced
```

**Conclusion:** The `protocolFeeReceiver` receives the full `creditToYield`, not just the fee.

---

## Recommendation

Update the `repay()` function to calculate and transfer only the proper protocol fee:

### Fix Snippet

```solidity
uint256 fee = creditToYield * protocolFee / BPS;
TokenUtils.safeTransfer(yieldToken, protocolFeeReceiver, fee);
```

Ensure this logic mirrors how fee deductions are handled in other functions like `redeem()`.
