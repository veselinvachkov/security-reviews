## [H-1] Fee Bypass Vulnerability in EbtcBSM

## Summary
The EbtcBSM contract contains a vulnerability in its fee calculation logic (_feeToBuy and _feeToSell) that allows users to bypass fees when buying or selling small amounts of assets. This can lead to a significant loss of revenue for the protocol.

## Finding Description
The _feeToBuy and _feeToSell functions calculate fees based on the provided amount and the configured basis points (feeToBuyBPS and feeToSellBPS). However, due to integer division, small amounts can result in zero fees.

### 1. _feeToBuy:
```solidity
return (_amount * feeToBuyBPS) / BPS;
```
If `_amount * feeToBuyBPS` is less than `BPS`, the result will be truncated to 0. For example, if `feeToBuyBPS` is set to 100 (1%), `_amount` can be up to 99 before a fee is applied.

### 2. _feeToSell:
```solidity
return (_amount * feeToSellBPS) / (feeToSellBPS + BPS);
```
Similar to `_feeToBuy`, small amounts can result in zero fees. For example, with `feeToSellBPS` set to `MAX_FEE` (2000), `_amount` can be up to 5 before a fee is applied.

The scenarios given above are only **EXAMPLES**.

This allows malicious users to repeatedly buy or sell small amounts, effectively bypassing fees and not contributing to the protocol's revenue.

## Impact Explanation
The impact is **high**. The protocol will lose revenue as users exploit this vulnerability. Over time, this can accumulate to a substantial amount. The revenue that should have been collected as fees will be missing. This also creates an unfair environment for other users who pay the intended fees.

## Likelihood Explanation
The likelihood is **high**. The vulnerability is easily exploitable by any user. The calculation to determine the maximum amount to bypass fees is straightforward. Given the public nature of the contract's code, everybody can see the `feeToBuyBPS` and `feeToSellBPS` and make the easy calculation.

## Proof of Concept
This test function can directly be added to `BuyAssetTests`:

```solidity
function testBuyAssetAndMakeItOutWith0Fee() public {
    // 1% fee (feeToBuyBPS = 100)
    vm.prank(techOpsMultisig);
    vm.expectEmit(false, true, false, false);
    emit FeeToBuyUpdated(0, 100);
    bsmTester.setFeeToBuy(100);

    vm.prank(testMinter);
    bsmTester.sellAsset(5e18, testMinter, 0);

    assertEq(mockAssetToken.balanceOf(testBuyer), 0);
    assertEq(mockEbtcToken.balanceOf(testBuyer), 10e18);

    uint256 buyerEbtcBalance = mockEbtcToken.balanceOf(testBuyer);
    uint256 initialGas = gasleft();

    uint256 i = 1;
    while (i <= 1000) {
        vm.prank(testBuyer);
        assertEq(bsmTester.buyAsset(99, testBuyer, 0), 99);
        i++;
    }

    uint256 finalGas = gasleft();
    uint256 totalGasUsed = initialGas - finalGas;

    buyerEbtcBalance = mockEbtcToken.balanceOf(testBuyer);
    console.log("Remaining ebtc:", buyerEbtcBalance);
    console.log("Total Gas Used:", totalGasUsed);
}
```

### Results:
- The test successfully buys **99,000 assets (99 * 1000)** without any fees being charged.
- The total gas used is **15,784,736**.
- This is still much better for the user than paying **5% on deposit and 5% on withdrawal**.

## Recommendation
### 1. Adjust Fee Calculation:
Modify the `_feeToBuy` and `_feeToSell` functions to ensure that even small amounts incur a proportional fee.
Consider using a minimum fee or a more precise calculation method that avoids truncation.

### 2. Implement Minimum Fee:
Implement a **minimum fee amount** for all transactions, ensuring that even very small transactions contribute to protocol revenue.

### 3. Add a Check to Prevent Small Transactions:
Add a `require` statement to the `buyAsset` and `sellAsset` functions to prevent transactions below a certain amount.

### Example of Adjusted _feeToBuy (using a minimum fee):
```solidity
function _feeToBuy(uint256 _amount) private view returns (uint256) {
    uint256 fee = (_amount * feeToBuyBPS) / BPS;
    uint256 minimumFee = 1; // Set a minimum fee

    if (fee == 0 && _amount > 0) {
        return minimumFee;
    } else {
        return fee;
    }
}
```
This ensures that even if the calculated fee is zero due to truncation, a **minimum fee of 1** is applied. Similar adjustments should be made to `_feeToSell`.


## [H-2] _ensureLiquidity uses super._totalBalance() wrong, leading to user funds mismanagement

# Summary
The `_ensureLiquidity` function in `ERC4626Escrow` is incorrect, violating the correlation between `_totalBalance() >= totalAssetsDeposited`, and user funds are being sent to `FEE_COLLECTOR`. This happens because in `_ensureLiquidity`, `super.feeProfit()` must be called instead of `super._totalBalance()`.

## Finding Description
The `ERC4626Escrow` contract overrides `_withdrawProfit` and `_onWithdraw` from `BaseEscrow`. However, when `_withdrawProfit` is called inside the function, there is a call to `_ensureLiquidity`. The problem arises here, in this line of code:

```solidity
uint256 liquidBalance = super._totalBalance();
```

The `super` call goes into `BaseEscrow`, where the implementation for `_totalBalance` is:

```solidity
function _totalBalance() internal virtual view returns (uint256) {
    return ASSET_TOKEN.balanceOf(address(this));
}
```

The total balance returned includes the total deposited `ASSET_TOKEN` plus fees. This means that if `_ensureLiquidity` is called with an amount greater than the fees, deposited asset tokens belonging to users will be transferred to the `FEE_RECIPIENT`.

```solidity
function _withdrawProfit(uint256 _profitAmount) internal override {
    uint256 redeemedAmount = _ensureLiquidity(_profitAmount);
    super._withdrawProfit(redeemedAmount);
}
```

### Funds Violation
If `redeemedAmount`, passed to `_withdrawProfit`, exceeds `feeProfit()`, the code executes:

```solidity
super._withdrawProfit(redeemedAmount);
```

This transfers more tokens than the accumulated fee profit, leading to a broken correlation between `_totalBalance()` and `totalAssetsDeposited`. As a result, `totalAssetsDeposited` becomes greater than `_totalBalance()`, indicating that the contract's accounting is incorrect and attempting to transfer funds that are not considered profit.

The `_claimProfit()` function in `BaseEscrow` ensures that only fee profits are withdrawn through its internal checks. However, directly calling `_withdrawProfit` bypasses these checks, leading to uncontrolled asset transfers.

```solidity
function _withdrawProfit(uint256 _profitAmount) internal virtual {
    ASSET_TOKEN.safeTransfer(FEE_RECIPIENT, _profitAmount);
}
```

```solidity
/// @notice Internal function to claim profits generated from fees and external lending
function _claimProfit() internal {
    uint256 profit = feeProfit();
    if (profit > 0) {
        _withdrawProfit(profit);
        // INVARIANT: total balance must be >= deposit amount
        require(_totalBalance() >= totalAssetsDeposited);
    }
}
```

This breaks the security guarantee of maintaining accurate asset balances and ensuring `_totalBalance()` reflects the deposited assets plus accumulated profit.

## Impact Explanation
The primary impact is the potential for financial loss. If `_withdrawProfit` is called incorrectly, it can transfer funds that do not belong to the intended recipient (`FEE_RECIPIENT`). This can result in a direct loss of assets, as users could lose funds due to the contract transferring assets that are not considered profit.

## Likelihood Explanation
This issue will occur whenever `_withdrawProfit` is called in `ERC4626Escrow`. Given that profit withdrawals are a core function of the contract, this scenario is reasonably probable during normal operation.

## Proof of Concept
A test scenario was created where `_withdrawProfit` is called. It shows that in the end, `totalAssetsDeposited` has a bigger value than `_totalBalance()`, which breaks the `BaseEscrow` logic and sends user funds to `FEE_COLLECTOR`.

### Test Contracts
#### TestERC4626Escrow
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.25;

import "@openzeppelin/contracts/mocks/token/ERC4626Mock.sol";
import {ERC4626Escrow} from "../src/ERC4626Escrow.sol";
import {BaseEscrow} from "../src/BaseEscrow.sol";

contract TestERC4626Escrow is ERC4626Escrow {
    constructor(address _externalVault, address _assetToken, address _bsm, address _governance, address _feeRecipient)
        ERC4626Escrow(_externalVault, _assetToken, _bsm, _governance, _feeRecipient)
    {}

    function testWithdrawProfit(uint256 _profitAmount) external {
        _withdrawProfit(_profitAmount);
    }
}
```

#### ERC4626EscrowTest
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.25;

import "forge-std/Test.sol";
import "@openzeppelin/contracts/mocks/token/ERC20Mock.sol";
import "@openzeppelin/contracts/mocks/token/ERC4626Mock.sol";
import {TestERC4626Escrow} from "test/TestERC4626Escrow.sol";
import {BaseEscrow} from "../src/BaseEscrow.sol";
import {AuthNoOwner} from "../src/Dependencies/AuthNoOwner.sol";

contract ERC4626EscrowTest is Test {
    ERC20Mock public mockAssetToken;
    ERC4626Mock public newExternalVault;
    TestERC4626Escrow public escrow;
    AuthNoOwner public bsmTester;

    function setUp() public {
        mockAssetToken = new ERC20Mock();
        newExternalVault = new ERC4626Mock(address(mockAssetToken));
        bsmTester = new AuthNoOwner();

        escrow = new TestERC4626Escrow(
            address(newExternalVault),
            address(mockAssetToken),
            address(bsmTester),
            address(bsmTester.authority()),
            address(this) // Using 'this' as fee recipient for testing
        );
    }

    function testTransferMoreFeesIntoFEE_RECIPIENTThanFeesAmount() public {
        // Deploy TestERC4626Escrow
        TestERC4626Escrow testEscrow = new TestERC4626Escrow(
            address(newExternalVault),
            address(mockAssetToken),
            address(bsmTester),
            address(bsmTester.authority()),
            address(escrow.FEE_RECIPIENT())
        );

        // Set testEscrow.totalAssetsDeposited to 1000000
        uint256 newTotalAssetsDeposited = 1_000_000;

        vm.prank(address(bsmTester));
        testEscrow.onMigrateTarget(newTotalAssetsDeposited);

        // Add 100000 tokens to testEscrow.totalBalance()
        uint256 additionalFunds = 100_000;
        mockAssetToken.mint(address(this), additionalFunds + newTotalAssetsDeposited);
        mockAssetToken.transfer(address(testEscrow), newTotalAssetsDeposited);
        mockAssetToken.transfer(address(testEscrow), additionalFunds);

        testEscrow.testWithdrawProfit(200000);

        assert(testEscrow.totalBalance() < testEscrow.totalAssetsDeposited());
    }
}
```

### Recommendation
Replace the call to `super._withdrawProfit(redeemedAmount)` with a call to `_claimProfit()` in `ERC4626Escrow` and use `super.feeProfit()` instead of `super._totalBalance()`. This ensures profit withdrawal is handled correctly and maintains the contract's invariant.
