**Title: Pike Finance**

Auditor: **vesko210**

## Issues found
|Severtity|Number of issues found
| ------- | -------------------- |
| High    | 3                    |
| Medium  | 5                    |
| Low     | 1                    |
| Info    | 1                    |

# Findings

## [H-1] Insolvent positions can not be liquidated causing bad debt to the protocol

# Summary
The protocol lacks a mechanism to handle positions that become deeply insolvent before liquidation occurs, potentially leading to unrecoverable bad debt in the lending system.

## Finding Description
In `PTokenModule::liquidateBorrowFresh`, after repaying the borrower’s debt and computing how many collateral tokens to seize, the code does.

Here `liquidateCalculateSeizeTokens` the seize amount is calculated as this:

```

// seizeTokens = actualRepayAmount \* (liquidationIncentive \* priceBorrowed) / (priceCollateral \* exchangeRate)

````

Then there is this check inside `liquidateBorrowFresh`:

```solidity
/* Revert if borrower collateral token balance < seizeTokens */
require(
    pTokenCollateral.balanceOf(borrower) >= seizeTokens,
    PTokenError.LiquidateSeizeTooMuch()
);
```

If, due to a sudden oracle price update or liquidators inactivity, the required `seizeTokens` is larger than the borrower’s balance, this `require` triggers and reverts the entire transaction. As a result:

* The borrower’s debt repayment is rolled back.
* No collateral is seized.
* The borrower remains under-collateralized with no mechanism to handle it.

## Impact Explanation

**High severity**: If positions cannot be liquidated due to insufficient collateral value, they remain in the system with debt that can never be repaid. This creates bad debt for the protocol.

## Likelihood Explanation

**High likelihood** in volatile market conditions. Crypto markets are known for sudden price crashes where assets can lose significant value in minutes.

* If liquidators cannot act quickly enough, or
* if network congestion prevents timely liquidation, or
* if position value drops to the point liquidation incentive outweighs the gas costs

then positions can rapidly become so underwater that they have insufficient collateral to cover the debt.

## Proof of Concept

The test shows that bad debt can be accrued and there is no way to deal with it.

Add the test inside `pToken.t.sol` and run:
`forge test --match-test test_badDebt_accrual -vvvv`

```solidity
function test_badDebt_accrual() public {
        address user1 = makeAddr("user1");
        address depositor = makeAddr("depositor");
        address liquidator = makeAddr("liquidator");

        vm.stopPrank();

        /// provide liquidity
        doDeposit(depositor, depositor, address(pUSDC), 1450e6);

        doDepositAndEnter(user1, user1, address(pWETH), 1e18);
        doBorrow(user1, user1, address(pUSDC), 1450e6);

        // NOTE this does change during normal protocol operations 
        // The liq treshold and also the colaterazation rate are important here
        // In some situations the price drop can be much smaller than in the current state, to start accumulating bad debt

        // 1450 / 0.825(weth liq threshold) = 1757.57 is liquidation threshold price for collateral
        mockOracle.setPrice(address(pWETH), 750e6, 18);

        LiquidationParams memory lp = LiquidationParams({
            prankAddress: liquidator,
            userToLiquidate: user1,
            collateralPToken: address(pWETH),
            borrowedPToken: address(pUSDC),
            repayAmount: 725e6,
            expectRevert: true,
            error: "LiquidateSeizeTooMuch()"
        });

        doLiquidate(lp);
    }
```

## Recommendation

Implement a mechanism to handle insolvent positions by allowing partial debt repayment during liquidation, with the remaining debt being socialized or covered by protocol reserves. Additionally, consider adding an emergency function that allows the protocol to close severely underwater positions at a loss to prevent accumulation of bad debt.



## [H-2] FallbackOracle price can cause unintended reverts or return a bad price

# Summary
The protocol’s multi‐oracle price lookup seames to be trusting more the fallbackOracle rather than the mainOracle without applying the same price checks as the primary (boundaries checks). As a result, a misbehaving fallback can both trigger an unnecessary revert even when the main oracle is healthy, and second it can return an unchecked price when the main oracle is down.

## Finding Description
The `OracleEngine::getPrice(asset)` function tries the `mainOracle`, then the `fallbackOracle` in two separate try/catch blocks. However:

### Unchecked Fallback can cause a revert even on mainOracle Success
```solidity
try IOracleProvider(config.mainOracle).getPrice(asset) returns (
    uint256 mainOraclePrice
) {
    if (config.fallbackOracle == address(0)) {
        return mainOraclePrice;
    }

    try IOracleProvider(config.fallbackOracle).getPrice(asset) returns (
        uint256 fallbackOraclePrice
    ) {
        if (config.lowerBoundRatio != 0 && config.upperBoundRatio != 0) {
            uint256 lowerBound =
                fallbackOraclePrice * config.lowerBoundRatio / 1e18;
            uint256 upperBound =
                fallbackOraclePrice * config.upperBoundRatio / 1e18;

            if (mainOraclePrice < lowerBound || mainOraclePrice > upperBound) {
                revert BoundValidationFailed();
            }
        }

        return mainOraclePrice;
    } catch {
        return mainOraclePrice;
    }
}
```

If the fallback’s `getPrice` succeeds and returns a incorrect `fallbackOraclePrice`, the code uses it to validate `mainPrice` bounds without verifying that `fallbackOraclePrice` itself. A bad fallback price can trigger `BoundValidationFailed()` even if the `mainOracle` is correct.

### Blind fallbackOracle price is taken as valid when mainOracle fails

```solidity
if (config.fallbackOracle != address(0)) {
    try IOracleProvider(config.fallbackOracle).getPrice(asset) returns (
        uint256 fallbackOraclePrice
    ) {
        return fallbackOraclePrice;
    } catch {
        revert InvalidFallbackOraclePrice();
    }
} else {
    revert InvalidMainOraclePrice();
}
```

When the main oracle fails, the fallback’s price is returned directly with no bound validation.

## Impact Explanation

**High**

* A stale or manipulated fallback price can slip through unchecked.
* A fallback that temporarily returns invalid price can cause the primary Oracle price to revert.
* The system cannot tolerate brief fallback feed failures without risking reverts on perfectly valid main prices.

Keep in mind reverting a `getPrice` functionality in a lending protocol can do much harm, accumulate bad debt, inability to repay pssitions before thay go underwater and so on.

## Likelihood Explanation

**Medium**: Тhe `fallbackOracle` has to return a invalide price that is > 0. As we use `mainOracle` for a more accurate price. We know that the fallback price is less secure than the main, so we can expect some inacuraties there.

## Proof of Concept

The test shows that:

* Totaly wrong price form the `fallbackOracle` is taken without validation of reasonable price boundaries.
* The `fallbackOracle` returns a invalid price which causes the `mainOracle` to revert when it is valid.

To run the test place it in `LocalOracle.t.sol`.
Run:
`forge test --match-test testOracleEngine_badFallBackOracleImplementation -vvvv`.

```solidity
function testOracleEngine_badFallBackOracleImplementation() public {
        uint256 wethPrice;
        vm.startPrank(_testState.admin);

        chainlinkOracleProvider.setAssetConfig(weth, wethPriceFeed, 1 hours);

        // get fallback if main oracle is down
        wethPriceFeed.setRoundData(2002e8, block.timestamp - 2 hours, block.timestamp - 2 hours);

        pythOracleProvider.setAssetConfig(weth, "weth", 0, 1 hours);
        pyth.setData(20e8, 0, -8);

        // NOTE the totaly wrong price form the fallbackOracle is taken without validation for and boundaries. 
        // 99% drop in price
        oracleEngine.setAssetConfig(weth, address(chainlinkOracleProvider), address(pythOracleProvider), 0.99e18, 1.01e18);
        wethPrice = oracleEngine.getPrice(weth);
        assertEq(wethPrice, 20e18);

        pyth.setData(2001e8, 0, -8);
        wethPriceFeed.setRoundData(2002e8, block.timestamp, block.timestamp);

        // verify bounds if set
        oracleEngine.setAssetConfig(
            weth,
            address(chainlinkOracleProvider),
            address(pythOracleProvider),
            0.99e18,
            1.01e18
        );

        wethPrice = oracleEngine.getPrice(weth);
        assertEq(wethPrice, 2002e18);

        // NOTE the fallbackOracle returns a invalid price which causes the mainOracle to revert when it is valid
        pyth.setData(4001e8, 0, -8);
        vm.expectRevert("BoundValidationFailed()");
        wethPrice = oracleEngine.getPrice(weth);
    }
```

## Recommendation

* Remove the boundary check that compares the `mainOracle` price against the `fallbackOracle` price. In nearly all cases, the `mainOracle` is expected to be more accurate. The current boundary check can introduce unnecessary failures or inconsistencies, especially when the fallback source is less reliable.
* When the `mainOracle` is down, introduce a variable to store the last known good price. This value can then be used to define acceptable bounds for the `fallbackOracle` price, rather than returning the fallback price blindly, as is currently done.



## [H-3] Liquidating or repaying small amounts can DoS both debt repayments and liquidations.

# Summary
The `PTokenModule::liquidateBorrow()` function allows liquidators to liquidate a borrower by paying some amount of their debt. This can be front-run by another liquidator who can repay as small as 1 wei and stop the liquidation. The same can happen if a user wants to repay all their debt but an attacker calls `repayBorrowOnBehalfOf` and repays just 1 wei of the debt, causing the user’s transaction to fail.

## Finding Description
When the liquidator wants to liquidate a position they set a `repayAmount` that they want to repay. If the `repayAmount` is the full debt amount, this can be stopped.  
The attacker can either repay or liquidate a tiny portion of the debt. This way, when the liquidator transaction is executed, it will revert due to the check `borrowBalance >= repayAmount`.

```solidity
function liquidateBorrowAllowed(
        address pTokenBorrowed,
        address pTokenCollateral,
        address borrower,
@>      uint256 repayAmount
    ) external view returns (RiskEngineError.Error) {
        RiskEngineData storage $ = _getRiskEngineStorage();

        if (!$.markets[pTokenBorrowed].isListed || !$.markets[pTokenCollateral].isListed)
        {
            return RiskEngineError.Error.MARKET_NOT_LISTED;
        }

@>      uint256 borrowBalance = IPToken(pTokenBorrowed).borrowBalanceStored(borrower);

        /* allow accounts to be liquidated if the market is deprecated */
        if (isDeprecated(IPToken(pTokenBorrowed))) {
@>          require(borrowBalance >= repayAmount, RiskEngineError.RepayMoreThanBorrowed());
        } else {
            /* The borrower must have shortfall in order to be liquidatable */
            (RiskEngineError.Error err,, uint256 shortfall) =
                getAccountLiquidityInternal(borrower);
            if (err != RiskEngineError.Error.NO_ERROR) {
                return err;
            }

            if (shortfall == 0) {
                return RiskEngineError.Error.INSUFFICIENT_SHORTFALL;
            }

            /* The liquidator may not repay more than what is allowed by the closeFactor */
            uint256 maxClose = $.closeFactorMantissa[pTokenBorrowed].toExp()
                .mul_ScalarTruncate(borrowBalance);
            if (repayAmount > maxClose) {
                return RiskEngineError.Error.TOO_MUCH_REPAY;
            }
        }
        return RiskEngineError.Error.NO_ERROR;
    }
```

## Impact Explanation

**High**

* If a user wants to repay their whole debt, an attacker can call `repayBorrowOnBehalfOf` and repay just 1 wei, which will cause the user’s transaction to fail. Repayment is a time-sensitive function. DoS’ing such a transaction might cause the attacked user to be liquidated, or incur interest expenses.
* If a liquidator wants to liquidate the whole position of a user, this can be front-run, and an attacker can liquidate or repay 1 wei of the debt, causing the full liquidation to revert.

## Likelihood Explanation

**Medium** — This can only be triggered (without significant costs for the exploiter) on full repayments.

## Proof of Concept

The test shows that a malicious user can block a full liquidation of a position by liquidating 1 wei of the position.
To run the test:
`forge test --match-test test_stop_liquidation_with_dustLiquidation -vvvv`

```solidity
function test_stop_liquidation_with_dustLiquidation() public {
    address user1      = makeAddr("user1");
    address depositor = makeAddr("depositor");
    address liquidator = makeAddr("liquidator");

    doDeposit(depositor, depositor, address(pUSDC), 2000e6);

    doDepositAndEnter(user1, user1, address(pWETH), 1e18);
    doBorrow(user1, user1, address(pUSDC), 1450e6);

    mockOracle.setPrice(address(pWETH), 1757e6, 18);
    
    LiquidationParams memory lp = LiquidationParams({
        prankAddress: liquidator,
        userToLiquidate: user1,
        collateralPToken: address(pWETH),
        borrowedPToken: address(pUSDC),
        repayAmount: 725e6,
        // We do expect the real liquidation to fail
        expectRevert: true,
        error: abi.encodePacked(bytes4(0xd1192049), uint256(6))
    });

    LiquidationParams memory lp1 = LiquidationParams({
        prankAddress: liquidator,
        userToLiquidate: user1,
        collateralPToken: address(pWETH),
        borrowedPToken: address(pUSDC),
        repayAmount: 1,
        expectRevert: false,
        error: abi.encodePacked(bytes4(0xd1192049), uint256(5))
    });

    doLiquidate(lp1);

    doLiquidate(lp);
}
```

## Recommendation

Introduce a minimum amount that can be repaid on behalf of the user and a minimum amount that can be liquidated from the user's total debt.



## [H-4] Borrowers can front-run and borrow cash to block emergency reserve withdrawals

# Summary
A malicious or ordinary borrower can borrow enough underlying assets, depleting the protocol’s cash, and thereby cause `reduceReservesEmergency`, `reduceReservesOwner`, `reduceReservesConfigurator` to revert due to insufficient balance.

## Finding Description
The PToken’s `borrowFresh` path enforces a cash check (`require(getCash() ≥ borrowAmount)`). However, the check is wrong as this allows the users to borrow from the reserve amount. This can create a DoS of the three permisioned functions stated in the summary.

- Permissioned address calls either of these functions (`reduceReservesEmergency`, `reduceReservesOwner`, `reduceReservesConfigurator`).  
- Attacker front-runs the above functions, executes the borrow, draining the contract’s cash.  
- The permissioned function fails.  
- There is a risk that the reserve is not going to be able to be claimed.  

```solidity
function borrowFresh(address borrower, address onBehalfOf, uint256 borrowAmount)
        internal
    {
    
        // code...

        /* Fail gracefully if protocol has insufficient underlying cash */
        // REVIEW This should deduct the reserve asets from getCash
@>      require(getCash() >= borrowAmount, PTokenError.BorrowCashNotAvailable());

        // code...
    }

function _reduceReserves(uint256 reduceAmount, uint256 totalReserve)
        internal
        returns (uint256 newReserve)
    {
        accrueInterest();
        // newReserve = totalReserves - reduceAmount

        // Verify market's block timestamp equals current block timestamp
        require(
            _getPTokenStorage().accrualBlockTimestamp == _getBlockTimestamp(),
            PTokenError.ReduceReservesFreshCheck()
        );

        // Fail gracefully if protocol has insufficient underlying cash
@>      require(getCash() >= reduceAmount, PTokenError.ReduceReservesCashNotAvailable());

        // Check reduceAmount ≤ reserves[n] (totalReserves)
        require(reduceAmount <= totalReserve, PTokenError.ReduceReservesCashValidation());

        /////////////////////////
        // EFFECTS & INTERACTIONS
        // (No safe failures beyond this point)

        newReserve = totalReserve - reduceAmount;

        // doTransferOut reverts if anything goes wrong, since we can't be sure if side effects occurred.
        doTransferOut(msg.sender, reduceAmount);
    }
```

## Impact Explanation

**Impact:** High

Once cash is borrowed, the protocol owner or emergency withdrawer cannot reduce or reallocate reserves, stopping any urgent liquidity management.

## Likelihood Explanation

**Likelihood:** High

Any borrower with sufficient collateral and borrow allowance can trigger this scenario without special permissions.

## Proof of Concept

The following test demonstrates that after a borrow drains cash, `reduceReservesEmergency` reverts.

To run the test:
`forge test --match-test test_ReservesEmergency_Attack -vvvv`

```solidity
function test_ReservesEmergency_Attack() public {
        address user1 = makeAddr("user1");
        uint256 value = 1e18;
        address depositor = makeAddr("depositor");

        doDeposit(depositor, depositor, address(pWETH), 1e18);

        deal(pUSDC.asset(), user1, value);

        uint256 reserveBefore = pUSDC.totalReserves();

        vm.startPrank(user1);
        IERC20(pUSDC.asset()).approve(address(pUSDC), value);
        pUSDC.addReserves(value);
        vm.stopPrank();

        uint256 reserveAfter = pUSDC.totalReserves();

        assertEq(reserveBefore + value, reserveAfter);
 
        // Now this is a frot-run of the EMERGANCY WITHDRAW and it works because it gets tokens from the reserve
        doBorrow(depositor, depositor, address(pUSDC), 1400e6);

        // We expect this to revert as the borrower has taken the cash and now there is no enought reserve
        vm.startPrank(getAdmin());
        vm.expectRevert();
        pUSDC.reduceReservesEmergency(value);
    }
```

## Recommendation

When a user borrows, the `maxBorrow` amount should be `getCash() - $.totalReserves`.

```diff
function borrowFresh(address borrower, address onBehalfOf, uint256 borrowAmount)
        internal
    {
        // code...
        /* Fail gracefully if protocol has insufficient underlying cash */
        // REVIEW shoudn't this deduct the reserve asets
-       require(getCash() >= borrowAmount, PTokenError.BorrowCashNotAvailable());
+       require(getCash() - $.totalReserves >= borrowAmount, PTokenError.BorrowCashNotAvailable());
        // code...
    }
```



## [M-1] There is no buffer enforced between the collateralFactorMantissa and liquidationThresholdMantissa

# Summary
The _CONFIGURATOR_PERMISSION role can unintentionally cause liquidations by changing market configurations.

## Finding Description
When a permissioned user uses the configureMarket, they are allowed to set both liquidationThresholdMantissa and collateralFactorMantissa. The only enforced constraint is that baseConfig.liquidationThresholdMantissa >= baseConfig.collateralFactorMantissa.

However, since no buffer is required between these two values, setting them too close—or equal—can immediately put a borrower at risk. If a user opens a position using the maximum borrowable amount (as allowed via the UI) and the market experiences even a minimal price fluctuation, their position may become eligible for liquidation right away.

For example:

- liquidationThresholdMantissa = 0.90  
- collateralFactorMantissa = 0.90  
- A user borrows the maximum allowed amount via the UI  
- A minimal price movement occurs → the user gets liquidated  

```solidity
function configureMarket(IPToken pToken, BaseConfiguration calldata baseConfig)
        external
    {
        checkPermission(_CONFIGURATOR_PERMISSION, msg.sender);
        RiskEngineData storage $ = _getRiskEngineStorage();
        // Verify market is listed
        require($.markets[address(pToken)].isListed, RiskEngineError.MarketNotListed());

        // Check collateral factor <= 0.9
        require(
            baseConfig.collateralFactorMantissa.toExp().lessThanOrEqualExp(
                _COLLATERAL_FACTOR_MAX_MANTISSA.toExp()
            ),
            RiskEngineError.InvalidCollateralFactor()
        );

        require(
            baseConfig.liquidationIncentiveMantissa >= _MANTISSA_ONE,
            RiskEngineError.InvalidIncentiveThreshold()
        );

        // Ensure that liquidation threshold <= 1
        require(
            baseConfig.liquidationThresholdMantissa <= _MANTISSA_ONE,
            RiskEngineError.InvalidLiquidationThreshold()
        );

        // Ensure that liquidation threshold >= CF
        require(
@>          baseConfig.liquidationThresholdMantissa >= baseConfig.collateralFactorMantissa,
            RiskEngineError.InvalidLiquidationThreshold()
        );

        BaseConfiguration memory oldConfiguration =
            $.markets[address(pToken)].baseConfiguration;
        BaseConfiguration storage baseConfiguration =
            $.markets[address(pToken)].baseConfiguration;

        // write new values
        baseConfiguration.collateralFactorMantissa = baseConfig.collateralFactorMantissa;
        baseConfiguration.liquidationThresholdMantissa =
            baseConfig.liquidationThresholdMantissa;
        baseConfiguration.liquidationIncentiveMantissa =
            baseConfig.liquidationIncentiveMantissa;

        // Emit event with asset, old collateral factor, and new collateral factor
        emit NewMarketConfiguration(pToken, oldConfiguration, baseConfig);
    }
```

## Impact Explanation

Users can get liquidated instantly after opening a position.

## Likelihood Explanation

All admin or permitioned function are considered low likelihood in the dox of Cantina. Importantly, this issue can also occur accidentally during regular protocol maintenance, without malicious intent.

## Proof of Concept

To run the test this code should be pased in `PToken.t.sol`. In the setUp i changed the paramenters of liquidationThresholdMantissa and collateralFactorMantissa. The setup should change like this:

```solidity
function setUp() public {
        
        //code...
        
        deployPToken("pike-usdc", "pUSDC", 6, 1e6, 0.9e18, 0.9e18, deployMockToken);
        deployPToken("pike-weth", "pWETH", 18, 2000e6, 0.9e18, 0.9e18, deployMockToken);

        //code...
    }
```

To run the test use:
`forge test --match-test test_instant_liquidation -vvvv`.

```solidity
function test_instant_liquidation() public {
    // Set up actor addresses for the scenario
    address user1      = makeAddr("user1");
    address attaker  = makeAddr("attaker");
    address liquidator = makeAddr("liquidator");

    // 1) Provide initial liquidity in USDC
    doDeposit(attaker, attaker, address(pUSDC), 2000e6);

    // 2) User1 opens a borrow position with WETH collateral
    doDepositAndEnter(user1, user1, address(pWETH), 1e18);
    // user1 then borrows 1,450 USDC against that collateral
    doBorrow(user1, user1, address(pUSDC), 1800e6);

    /// 3) Set price to the liquidation threshold
    mockOracle.setPrice(address(pWETH), 1999e6, 18);
    
    // 4) Prepare liquidation parameters
    LiquidationParams memory lp = LiquidationParams({
        prankAddress: liquidator,
        userToLiquidate: user1,
        collateralPToken: address(pWETH),
        borrowedPToken: address(pUSDC),
        repayAmount: 725e6,
        expectRevert: false,
        error: abi.encodePacked(bytes4(0xd1192049), uint256(5))
    });

    doLiquidate(lp);
}
```

## Recommendation

Implement a buffer and always configure your liquidationThresholdMantissa strictly above your collateralFactorMantissa. This is one way that this can be solved easily:

```diff
+ // 5% buffer between liquidationThreshold and collateralFactor
+ uint256 BUFFER = 0.05e18;

// code...

function configureMarket(IPToken pToken, BaseConfiguration calldata baseConfig)
        external
    {
        // code...
-       // Ensure that liquidation threshold >= CF
+       // Ensure that liquidation threshold >= CF + BUFFER
        require(
-           baseConfig.liquidationThresholdMantissa >= baseConfig.collateralFactorMantissa,
+           baseConfig.liquidationThresholdMantissa >= baseConfig.collateralFactorMantissa + BUFFER,
            RiskEngineError.InvalidLiquidationThreshold()
        );
        // code...
```



## [M-2] Attackers can create dust positions that are unprofitable to liquidate

# Summary
The borrow functionality lacks enforcement of a minimum borrow threshold, enabling malicious actors to generate numerous small ("dust") positions. These positions may not be worth the gas fees required to liquidate, ultimately putting the protocol at risk of accumulating unrecoverable bad debt.

## Issue Details
The protocol currently permits borrowing of arbitrarily small amounts. An attacker could exploit this by:

- Creating a large number of wallet addresses  
- Using each address to make small deposits and then borrow  
- Allowing these positions to slip underwater, knowing they won’t be economically attractive for liquidation due to gas costs and unprofitable reward  

## Impact
**Severity:** High — These unliquidated dust loans can build up over time and result in significant bad debt for the protocol.

## Likelihood
**Probability:** Low — Attackers need to create thousands of dust positions.

## Proof of Concept
The test shows that the gas cost to execute the liquidation is much greater than the reward that the liquidator gets after the liquidation is executed.  

Place the test inside `PToken.t.sol` and run:  
`forge test --match-test test_small_possitions_attack -vvvv`

```solidity
function test_small_possitions_attack() public {
    address user1      = makeAddr("user1");
    address depositor = makeAddr("depositor");
    address liquidator = makeAddr("liquidator");

    doDeposit(depositor, depositor, address(pUSDC), 2000e6);

    doDepositAndEnter(user1, user1, address(pWETH), 1e12);
    doBorrow(user1, user1, address(pUSDC), 1450);

    mockOracle.setPrice(address(pWETH), 1757e6, 18);
    
    LiquidationParams memory lp = LiquidationParams({
        prankAddress: liquidator,
        userToLiquidate: user1,
        collateralPToken: address(pWETH),
        borrowedPToken: address(pUSDC),
        repayAmount: 725,
        expectRevert: false,
        error: abi.encodePacked(bytes4(0xd1192049), uint256(5))
    });

    uint256 balBefore = pWETH.balanceOf(liquidator);

    uint256 gasBefore = gasleft();

    doLiquidate(lp);

    uint256 gasAfter = gasleft();

    uint256 balAfter = pWETH.balanceOf(liquidator);

    uint256 gasTakenInTheLiquidation = gasBefore - gasAfter;

    console.log(balBefore);

    console.log(balAfter);

    console.log(gasTakenInTheLiquidation);
}
```

## Recommendation

Introduce a minimum borrow (or deposit) limit to prevent creation of uneconomical positions. This ensures all positions are always worth liquidating if necessary.



## [M-3] Fee-on-Transfer tokens prevent full debt repayment and liquidation

# Summary
Fee-on-transfer tokens can prevent full debt repayment or liquidation, leaving residual “bad debt” that the protocol cannot clear.

## Finding Description
The `repayBorrowFresh` function uses `doTransferIn` to collect repayment amounts. For fee-on-transfer tokens, `doTransferIn` returns the net amount received (`actualRepayAmount`), which is less than the intended `repayAmountFinal`. The contract then deducts `actualRepayAmount` from the borrower’s balance:

```solidity
function repayBorrowFresh(address payer, address onBehalfOf, uint256 repayAmount)
        internal
        returns (uint256)
    {
        PTokenData storage $ = _getPTokenStorage();
        /* Fail if repayBorrow not allowed */
        RiskEngineError.Error allowed = $.riskEngine.repayBorrowAllowed(address(this));
        require(
            allowed == RiskEngineError.Error.NO_ERROR,
            PTokenError.RepayBorrowRiskEngineRejection(uint256(allowed))
        );

        /* Verify market's block timestamp equals current block timestamp */
        require(
            $.accrualBlockTimestamp == _getBlockTimestamp(),
            PTokenError.RepayBorrowFreshnessCheck()
        );

        /* We fetch the amount the borrower owes, with accumulated interest */
        uint256 accountBorrowsPrev = borrowBalanceStoredInternal(onBehalfOf);

        /* If repayAmount == type(uint256).max, repayAmount = accountBorrows */
        uint256 repayAmountFinal =
            repayAmount == type(uint256).max ? accountBorrowsPrev : repayAmount;

        /////////////////////////
        // EFFECTS & INTERACTIONS
        // (No safe failures beyond this point)

        /*
         * We call doTransferIn for the payer and the repayAmount
         *  On success, the pToken holds an additional repayAmount of cash.
         *  doTransferIn reverts if anything goes wrong, since we can't be sure if side effects occurred.
         *   it returns the amount actually transferred, in case of a fee.
         */
@>      uint256 actualRepayAmount = doTransferIn(payer, repayAmountFinal);

        /*
         * We calculate the new borrower and total borrow balances, failing on underflow:
         *  accountBorrowsNew = accountBorrows - actualRepayAmount
         *  totalBorrowsNew = totalBorrows - actualRepayAmount
         */
@>      uint256 accountBorrowsNew = accountBorrowsPrev - actualRepayAmount;
@>      uint256 totalBorrowsNew = $.totalBorrows - actualRepayAmount;

        /* We write the previously calculated values into storage */
        $.accountBorrows[onBehalfOf].principal = accountBorrowsNew;
        $.accountBorrows[onBehalfOf].interestIndex = $.borrowIndex;
        $.totalBorrows = totalBorrowsNew;

        $.riskEngine.repayBorrowVerify(IPToken(address(this)), onBehalfOf);

        /* We emit a RepayBorrow event */
        emit RepayBorrow(
            payer, onBehalfOf, actualRepayAmount, accountBorrowsNew, totalBorrowsNew
        );

        return actualRepayAmount;
    }
```

Because `actualRepayAmount < repayAmountFinal`, `accountBorrowsNew` remains > 0 even when attempting to clear the full debt (`repayAmount == type(uint256).max`). Even if liquidate or repay is called again the same issue is going to occure. And with time passing all those non-zero possitions are going to result in uncollectible bad debt accumulating in the protocol.

## Impact Explanation
**High**
- **Repayment finality:** Borrowers cannot fully repay outstanding debt.  
- **Liquidation completeness:** Liquidators cannot fully close undercollateralized positions.  
- **Protocol solvency:** Residual debt persists, risking protocol losses over time.  

## Likelihood Explanation
**Low** – Occurs only when a fee-on-transfer token is used.

## Proof of Concept
Simulate a 0.01% fee-on-transfer token by reducing `actualRepayAmount`, then observe leftover debt after liquidation:

```solidity
function simulateFeeOnTransfer_AndDeduct_AcctualAmountRepayed(uint256 amount)
        internal
        virtual
        returns (uint256)
    {
        return amount - amount * 9999 / 10000;
    }
    
function test_fee_when_liquidating_cause_leftover_debt() public {
    address user1      = makeAddr("user1");
    address depositor = makeAddr("depositor");
    address liquidator = makeAddr("liquidator");

    mockOracle.setPrice(address(pWETH), 2000e6, 18);
    doDeposit(depositor, depositor, address(pUSDC), 2000e6);
    
    doDepositAndEnter(user1, user1, address(pWETH), 1e18);
    doBorrow(user1, user1, address(pUSDC), 1450e6);
    mockOracle.setPrice(address(pWETH), 1757e6, 18);

    LiquidationParams memory lp = LiquidationParams({
    prankAddress: liquidator,
    userToLiquidate: user1,
    collateralPToken: address(pWETH),
    borrowedPToken: address(pUSDC),
    repayAmount: simulateFeeOnTransfer_AndDeduct_AcctualAmountRepayed(725e6),
    expectRevert: false,
    error: abi.encodePacked(bytes4(0xd1192049), uint256(6))
    });

    doLiquidate(lp);

    console.log(pUSDC.borrowBalanceCurrent(user1));

    }
```

```
console::log(1449927500 [1.449e9])
```

## Recommendation
Disallow fee-on-transfer tokens, or enforce repayment accounting by requiring the protocol to top-up fee shortfalls.



## [L-1] Global supply cap can be used to griefing and force liquidation

# Summary
The protocol’s global SupplyCap and BorrowCap can be used to harm users. The attacker has to front-run a user transaction and by depositing he can increase the supply close to the SupplyCap, causing honest user transactions (additional collateral deposits) to revert and thereby enabling forced liquidations.

## Finding Description
So basicly inside PTokenModule there are the functions deposit/mint which call mintFresh and inside mintFresh there is a check to see if a mint is allowed (function mintAllowed). And finally there is a check inside mintAllowed which checks wether the global SupplyCap has been excieded.

```solidity
function mintAllowed(address account, address pToken, uint256 mintAmount)
        external
        view
        returns (RiskEngineError.Error)
    {
        RiskEngineData storage $ = _getRiskEngineStorage();

        // Pausing is a very serious situation - we revert to sound the alarms
        require(!$.mintGuardianPaused[pToken], RiskEngineError.MintPaused());

        if (!$.markets[pToken].isListed) {
            return RiskEngineError.Error.MARKET_NOT_LISTED;
        }

        uint8 category = $.accountCategory[account];
        // Should check if account is in emode and pToken is allowed
        if (category != 0 && !$.emodes[category].allowed) {
            return RiskEngineError.Error.EMODE_NOT_ALLOWED;
        }

@>      uint256 cap = $.supplyCaps[pToken];
        // Skipping the cap check for uncapped coins to save some gas
        if (cap != type(uint256).max) {
            uint256 pTokenSupply = IPToken(pToken).totalSupply();

            uint256 nextTotalSupply = IPToken(pToken).exchangeRateStored().toExp()
                .mul_ScalarTruncateAddUInt(pTokenSupply, mintAmount);
@>          if (nextTotalSupply > cap) {
@>              return RiskEngineError.Error.SUPPLY_CAP_EXCEEDED;
            }
        }

        return RiskEngineError.Error.NO_ERROR;
    }
```

Now why is that a problem. Imagine this example:

* User needs to deposit collateral to avoid liquidation.
* Attacker sees the user’s pending “deposit” tx in the mempool.
* Attacker front-runs by minting up to the cap or close enough to it.
* User’s deposit now fails the exhausted-cap check and reverts.
* User remains under-collateralized and can be liquidated.

This allows a griefing attack leading to forced liquidations.

## Impact Explanation

**Impact:** High.

* Users can be induced to fail their own protective transactions, resulting in a loss via liquidation.
* Users lose autonomy to maintain their positions.
* Legitimate deposit/borrow operations can be arbitrarily blocked.

## Likelihood Explanation

**Likelihood:** Medium.

* The cap has to be different from `type(uint256).max`.
* No special privileges are required to mint/borrow up to the cap.
* A front-run is recuired with a fast calculation of how much should the attakerdeposit to be close enought to the cap so the user transaction reverts.

## Proof of Concept

To run the test, the cap should be changed, currently all test functions are useing a cap = `type(uint256).max`. To change that i went inside TestDeploy and on line 196 made cap = `10e18`. This is crutial for the test to work.
[https://cantina.xyz/code/a0806644-7d91-457a-a08d-aee2db73f352/pike-local-markets/test/helpers/TestDeploy.sol?lines=196,196](https://cantina.xyz/code/a0806644-7d91-457a-a08d-aee2db73f352/pike-local-markets/test/helpers/TestDeploy.sol?lines=196,196)

Now place the test inside LocalPToken and call it with:
`forge test --match-test test_attaker_blocks_user_collateral_deposit_User_gets_liquidated -vvvv`.

```solidity
function test_attaker_blocks_user_collateral_deposit_User_gets_liquidated() public {
    // Set up actor addresses for the scenario
    address user1      = makeAddr("user1");
    address attaker = makeAddr("attaker");
    address liquidator = makeAddr("liquidator");

    // 1) Provide initial liquidity in USDC
    doDeposit(attaker, attaker, address(pUSDC), 2000e6);

    // 2) User1 opens a borrow position with WETH collateral
    doDepositAndEnter(user1, user1, address(pWETH), 1e18);
    // user1 then borrows 1,450 USDC against that collateral
    doBorrow(user1, user1, address(pUSDC), 1450e6);

    /// 3) Set price to the liquidation threshold
    mockOracle.setPrice(address(pWETH), 1757e6, 18);
    
    // 4) Prepare liquidation parameters
    LiquidationParams memory lp = LiquidationParams({
        prankAddress: liquidator,
        userToLiquidate: user1,
        collateralPToken: address(pWETH),
        borrowedPToken: address(pUSDC),
        repayAmount: 725e6,
        expectRevert: false,
        error: abi.encodePacked(bytes4(0xd1192049), uint256(5))
    });

    // 5) attaker pushes the collateral close to it's cap
    doDeposit(attaker, attaker, address(pWETH), 8.5e18);

    /// 6) User1 is unable to deposit and save his borrow possition.
    // We can see that this call expects a revert, if the amount is less than 0.5 it is going to succed
    // as the cap is still not going to be excceded. The cap we set is 10e18 for simplisity.
    doAction(
        ActionParameters({
            action: Action.SUPPLY,
            pToken: address(pWETH),
            tokenAddress: IPToken(address(pWETH)).asset(),
            amount: 1e18,
            expectRevert: true,
            error: "MintRiskEngineRejection(7)",
            prankAddress: user1,
            onBehalfOf: user1
        })
    );

    // 7) Execute liquidation
    doLiquidate(lp);
}
```

## Recommendation

To mitigate the issue introduce per-user cap rather than globally or remove global cap entirely.



## [I-1] Redundant check can block liquidation attempt

# Summary
In `liquidateBorrowFresh`, the function rejects `repayAmount == type(uint256).max`, preventing liquidators from using the “max” shorthand to repay an entire debt, despite `repayBorrowFresh` internally supporting `repayAmount == type(uint256).max` by capping it to the borrower’s outstanding debt.

## Finding Description
```solidity
function liquidateBorrowFresh(
      address liquidator,
      address borrower,
      uint256 repayAmount,
      IPToken pTokenCollateral
  ) internal {
      /* Fail if liquidate not allowed */
      RiskEngineError.Error allowed = _getPTokenStorage()
          .riskEngine
          .liquidateBorrowAllowed(
          address(this), address(pTokenCollateral), borrower, repayAmount
      );
      require(
          allowed == RiskEngineError.Error.NO_ERROR,
          PTokenError.LiquidateRiskEngineRejection(uint256(allowed))
      );

      /* Verify market's block timestamp equals current block timestamp */
      require(
          _getPTokenStorage().accrualBlockTimestamp == _getBlockTimestamp(),
          PTokenError.LiquidateFreshnessCheck()
      );

      /* Verify pTokenCollateral market's block timestamp equals current block timestamp */
      require(
          pTokenCollateral.accrualBlockTimestamp() == _getBlockTimestamp(),
          PTokenError.LiquidateCollateralFreshnessCheck()
      );

      /* Fail if borrower = liquidator */
      require(borrower != liquidator, PTokenError.LiquidateLiquidatorIsBorrower());

      /* Fail if repayAmount = 0 */
      require(repayAmount != 0, PTokenError.LiquidateCloseAmountIsZero());

      /* Fail if repayAmount = type(uint256).max */
      require(
          repayAmount != type(uint256).max, PTokenError.LiquidateCloseAmountIsUintMax()
      );

      /* Fail if repayBorrow fails */
      uint256 actualRepayAmount = repayBorrowFresh(liquidator, borrower, repayAmount);

      /////////////////////////
      // EFFECTS & INTERACTIONS
      // (No safe failures beyond this point)

      /* We calculate the number of collateral tokens that will be seized */
      (RiskEngineError.Error amountSeizeError, uint256 seizeTokens) = _getPTokenStorage(
      ).riskEngine.liquidateCalculateSeizeTokens(
          borrower, address(this), address(pTokenCollateral), actualRepayAmount
      );
      require(
          amountSeizeError == RiskEngineError.Error.NO_ERROR,
          PTokenError.LiquidateCalculateAmountSeizeFailed(uint256(amountSeizeError))
      );

      /* Revert if borrower collateral token balance < seizeTokens */
      require(
          pTokenCollateral.balanceOf(borrower) >= seizeTokens,
          PTokenError.LiquidateSeizeTooMuch()
      );

      // If this is also the collateral, run seizeInternal to avoid re-entrancy, otherwise make an external call
      if (address(pTokenCollateral) == address(this)) {
          seizeInternal(address(this), liquidator, borrower, seizeTokens);
      } else {
          pTokenCollateral.seize(liquidator, borrower, seizeTokens);
      }

      /* We emit a LiquidateBorrow event */
      emit LiquidateBorrow(
          liquidator,
          borrower,
          actualRepayAmount,
          address(pTokenCollateral),
          seizeTokens
      );
  }
```

```solidity
function repayBorrowFresh(address payer, address onBehalfOf, uint256 repayAmount)
      internal
      returns (uint256)
  {
      PTokenData storage $ = _getPTokenStorage();
      /* Fail if repayBorrow not allowed */
      RiskEngineError.Error allowed = $.riskEngine.repayBorrowAllowed(address(this));
      require(
          allowed == RiskEngineError.Error.NO_ERROR,
          PTokenError.RepayBorrowRiskEngineRejection(uint256(allowed))
      );

      /* Verify market's block timestamp equals current block timestamp */
      require(
          $.accrualBlockTimestamp == _getBlockTimestamp(),
          PTokenError.RepayBorrowFreshnessCheck()
      );

      /* We fetch the amount the borrower owes, with accumulated interest */
      uint256 accountBorrowsPrev = borrowBalanceStoredInternal(onBehalfOf);

      /* If repayAmount == type(uint256).max, repayAmount = accountBorrows */
      uint256 repayAmountFinal = repayAmount == type(uint256).max ? accountBorrowsPrev : repayAmount;

      /////////////////////////
      // EFFECTS & INTERACTIONS
      // (No safe failures beyond this point)

      /*
       * We call doTransferIn for the payer and the repayAmount
       *  On success, the pToken holds an additional repayAmount of cash.
       *  doTransferIn reverts if anything goes wrong, since we can't be sure if side effects occurred.
       *   it returns the amount actually transferred, in case of a fee.
       */
      uint256 actualRepayAmount = doTransferIn(payer, repayAmountFinal);

      /*
       * We calculate the new borrower and total borrow balances, failing on underflow:
       *  accountBorrowsNew = accountBorrows - actualRepayAmount
       *  totalBorrowsNew = totalBorrows - actualRepayAmount
       */
      uint256 accountBorrowsNew = accountBorrowsPrev - actualRepayAmount;
      uint256 totalBorrowsNew = $.totalBorrows - actualRepayAmount;

      /* We write the previously calculated values into storage */
      $.accountBorrows[onBehalfOf].principal = accountBorrowsNew;
      $.accountBorrows[onBehalfOf].interestIndex = $.borrowIndex;
      $.totalBorrows = totalBorrowsNew;

      $.riskEngine.repayBorrowVerify(IPToken(address(this)), onBehalfOf);

      /* We emit a RepayBorrow event */
      emit RepayBorrow(
          payer, onBehalfOf, actualRepayAmount, accountBorrowsNew, totalBorrowsNew
      );

      return actualRepayAmount;
  }
```

This design lets callers pass `uint256.max` to indicate “repay full debt,” and the function safely caps it to the borrower’s current balance.

In `liquidateBorrowFresh`, immediately after validating inputs, there is:

```solidity
// Fail if repayAmount = type(uint256).max
require(repayAmount != type(uint256).max, PTokenError.LiquidateCloseAmountIsUintMax());
```

This causes any liquidation call that attempts to use the “max” shorthand to revert, even though the internal repay logic would correctly cap and process full repayment.

## Recommendation

Remove the redundant `require` that blocks `type(uint256).max` in `liquidateBorrowFresh`. Allow the internal capping logic in `repayBorrowFresh` to handle “max” amounts consistently for both user repayments and liquidation calls.

**Fixed Code Snippet:**

```solidity
function liquidateBorrowFresh(
     address liquidator,
     address borrower,
     uint256 repayAmount,
     IPToken pTokenCollateral
 ) internal {
     /* ... other checks ... */
-    /* Fail if repayAmount = type(uint256).max */
-    require(repayAmount != type(uint256).max, PTokenError.LiquidateCloseAmountIsUintMax());
     /* Perform the repay; internal logic caps max to borrower balance */
     uint256 actualRepayAmount = repayBorrowFresh(liquidator, borrower, repayAmount);
     /* ... continue liquidation ... */
 }
```