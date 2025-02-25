**Title: 2025-02-raac**

Auditor: **vesko210**

## Issues found
|Severtity|Number of issues found
| ------- | -------------------- |
| High    | 6                    |
| Medium  | 3                    |
| Info    | 1                    |

# Findings

## [H-1] Changing the liquidity pool address leads to stuck funds

## Summary

If this function is called more than one time without transferring funds from the old liquidity pool to the new one, any RAAC tokens held in the previous liquidity pool will become inaccessible. This leads to a permanent loss of funds.

## Vulnerability Details

* There is a scenario where `setLiquidityPool` is called by mistake or the pool address needs to be changed for security reasons and the `owner` calls the `setLiquidityPool` function to do so, but this results in the loss of all tokens.

### **Issue: Funds in the Old Liquidity Pool Are Not Transferred**

* The function `setLiquidityPool` allows the contract owner to change the liquidity pool address without ensuring that any existing RAAC token balance in the old liquidity pool is transferred to the new one.
* If `depositRAACFromPool` was previously used and RAAC tokens remain in the old liquidity pool, updating the liquidity pool address **does not** move those tokens.
* As a result, those tokens become **unrecoverable**, leading to financial loss.

### **Affected Code:**

```solidity
function setLiquidityPool(address _liquidityPool) external onlyOwner {
    liquidityPool = _liquidityPool;
    emit LiquidityPoolSet(_liquidityPool);
}
```

## Impact

* **Irrecoverable Tokens:** Since there is no built-in mechanism to reclaim tokens from the old liquidity pool, funds could be lost indefinitely.

## Tools Used

* **Manual Code Review**

## Recommendations

### **1. Implement a Safe Transfer Mechanism**

Before updating `liquidityPool`, ensure that the old pool’s RAAC balance is transferred to the new pool:

```Solidity
function setLiquidityPool(address _liquidityPool) external onlyOwner {
    require(_liquidityPool != address(0), "Invalid address");

    if (liquidityPool != address(0)) {
        uint256 balance = raacToken.balanceOf(liquidityPool);
        if (balance > 0) {
            raacToken.safeTransferFrom(liquidityPool, _liquidityPool, balance);
        }
    }

    liquidityPool = _liquidityPool;
    emit LiquidityPoolSet(_liquidityPool);
}
```

This ensures that any existing RAAC tokens in the old liquidity pool are **transferred to the new pool** before updating the address.

## Conclusion

The current implementation of `setLiquidityPool` introduces a **risk of fund loss** if called multiple times. Implementing a **safe transfer mechanism**will prevent funds from being lost when the liquidity pool address is changed.

## Severity

* The likelihood might be low but the impact is critical.


## [H-2] Fund mismanagement in Treasury

### Summary

The `allocateFunds` function incorrectly overwrites allocations instead of accumulating them, leading to fund mismanagement and potential financial discrepancies.

### Vulnerability Details

The current code would not be problematic if the `amount` to be allocated was calculated before the `allocateFunds` function was called. However, there is no such implementation, and the `amount` passed is never checked to see if it is updated with the user's current allocated funds. This results in overwriting, which would cause a loss of funds for the user.

* **Affected Function:** `Treasury:allocateFunds`
* **Issue:**

```Solidity
    /**
     * @notice Allocates funds to a recipient
     * @dev Only callable by accounts with ALLOCATOR_ROLE
     * Records allocation without transferring tokens
     * @param recipient Address to allocate funds to
     * @param amount Amount of funds to allocate
     */
    function allocateFunds(
        address recipient,
        uint256 amount
    ) external override onlyRole(ALLOCATOR_ROLE) {
        if (recipient == address(0)) revert InvalidRecipient();
        if (amount == 0) revert InvalidAmount();
        
       //@audit users allocation being overwritten rather than updated 
        _allocations[msg.sender][recipient] = amount;
        emit FundsAllocated(recipient, amount);
    }
```

This resets the allocations instead of adding or subtracting them by deleting the previous allocations.

### Impact

* Loss of previous allocations, affecting fund tracking
* Incorrect financial records and misallocations

### Recommendations

* **Modify the function to accumulate and also to subtract allocations.**

## Conclusion 
The current implementation of `allocateFunds` leads to unintended fund overwriting, resulting in financial inconsistencies. By modifying the function to properly adjust allocations instead of replacing them, the contract can ensure accurate fund tracking and prevent user losses. Implementing proper validation and accumulation logic will enhance security.


## [H-3] Unrestricted Boost Delegation

## Summary

In the `delegateBoost` function of the `BoostController` contract, where a user can delegate a boost to another address without updating their balance or the `userBoosts` mapping properly. This leads to a duplication of funds issue, allowing both the delegator and the recipient to utilize the same boost amount multiple times without restriction.

## Vulnerability Details

The `delegateBoost` function allows a user to delegate a boost to another address, storing the delegation details in the `userBoosts` mapping. However, the function does not update the delegator's balance or enforce any restrictions on repeated delegations. This results in a scenario where a user can delegate multiple times TO OTHER DIFFERENT ADDRESSES WITHOUT BOUNDARIES, effectively duplicating the amount of boost available in the system.

File location: 2025-02-raac\contracts\core\governance\boost\BoostController.sol

### Relevant Code:

```solidity
function delegateBoost(
    address to,
    uint256 amount,
    uint256 duration
) external override nonReentrant {
    if (paused()) revert EmergencyPaused();
    if (to == address(0)) revert InvalidPool();
    if (amount == 0) revert InvalidBoostAmount();
    if (duration < MIN_DELEGATION_DURATION || duration > MAX_DELEGATION_DURATION) 
        revert InvalidDelegationDuration();
    
    uint256 userBalance = IERC20(address(veToken)).balanceOf(msg.sender);
    if (userBalance < amount) revert InsufficientVeBalance();
    
    UserBoost storage delegation = userBoosts[msg.sender][to];
    if (delegation.amount > 0) revert BoostAlreadyDelegated();
    
    delegation.amount = amount;
    delegation.expiry = block.timestamp + duration;
    delegation.delegatedTo = to;
    delegation.lastUpdateTime = block.timestamp;
    
    emit BoostDelegated(msg.sender, to, amount, duration);
}
```

The following lines check if the delegated address has already been delegated and does not prevent from delegating more than one address. Also it does not check if the user has already delegated his entire `userBalance`.

```solidity
        uint256 userBalance = IERC20(address(veToken)).balanceOf(msg.sender);
        if (userBalance < amount) revert InsufficientVeBalance();
        
        UserBoost storage delegation = userBoosts[msg.sender][to];
        if (delegation.amount > 0) revert BoostAlreadyDelegated();
```

The function allows multiple delegations without properly updating the delegator's balance or preventing re-delegation. This results in an unlimited boost duplication exploit.

## Example of Exploit:

Since the contract **does not check if Alice has enough veRAAC to back all delegations**, Alice could theoretically:

* Delegate **10,000 veRAAC** to Bob
* Delegate **10,000 veRAAC** to Charlie
* Delegate **10,000 veRAAC** to Dave
* And so on…

Each recipient would **receive a boost as if the veRAAC were fully available**, even though Alice only has **10,000 veRAAC total**. This effectively **creates "fake" boost power** out of thin air.

## Impact

* **Boost Duplication:** A user can delegate more boost than they actually possess, leading to an inflation of the boost supply.
* **Exploitation for Unfair Advantage:** Malicious actors can repeatedly delegate their boost, leading to unfair allocation of rewards and disrupting the system's intended functionality.
* Pools may end up with more boost than actually exists, affecting calculations in critical parts of the protocol.

## Tools Used

Manual review

## Recommendations

* **Deduct Delegated Boost from the User’s Balance:** The contract should ensure that whenever a boost is delegated, the user’s effective balance is reduced accordingly.
* **Track and Prevent Multiple Delegations:** The contract should modify the logic to prevent a user from delegating multiple times without proper balance updates.
* **Add a mapping to track how much veRAAC a user has delegated in total**

## Severity

The vulnerability is of high severity as it enables users to delegate more boost than they actually possess, resulting in the inflation of the boost supply. This exploit can be leveraged by malicious actors to gain unfair advantages, distort reward distributions, and create significant imbalances within the protocol. The lack of proper checks on multiple delegations undermines the system's intended functionality. Consequently, this flaw poses a critical risk to the integrity and stability of the entire system.



## [H-4] Funds Locked Due to Misconfigured Treasury Address

## Summary

The function `applyTreasuryUpdate()` updates the `treasury` address to a new one after a timelock period. However, the function does not handle the transfer of collected fees from the old treasury to the new treasury before updating the address. This could lead to potential fund mismanagement if there are remaining balances in the old treasury that are not explicitly transferred to the new address before the update.

## Vulnerability Details

Code sniped is found in `2025-02-raac\contracts\core\collectors\FeeCollector.sol` on lines 306-312

```solidity
function applyTreasuryUpdate() external {
        if (pendingTreasury.newAddress == address(0)) revert InvalidAddress();
        if (block.timestamp < pendingTreasury.effectiveTime) revert UnauthorizedCaller();
        
        //@audit what is hapening with the treasury.balance and the fees collected there
        // all collected fees in the old treasury should be transferred to the new treasury
        // before the address is updated
        treasury = pendingTreasury.newAddress;
        delete pendingTreasury;
    }
```

## Impact

Loss or misallocation of protocol fees if the old treasury retains funds that should be used for protocol operations.

## Tools Used

Manual review

## Recommendations

To mitigate this issue, modify `applyTreasuryUpdate()` to include logic that transfers all collected fees from the old treasury to the new treasury before updating the address. That is completely necessary every time a treasury is changed (even if the contract uses/distributes the fees in another function).

Suggested implementation:

```solidity
function applyTreasuryUpdate() external {
    if (pendingTreasury.newAddress == address(0)) revert InvalidAddress();
    if (block.timestamp < pendingTreasury.effectiveTime) revert UnauthorizedCaller();
    
    // Ensure all collected fees are transferred before updating
    uint256 balance = raacToken.balanceOf(treasury);
    if (balance > 0) {
        raacToken.safeTransfer(pendingTreasury.newAddress, balance);
    }
    
    // Update treasury address
    treasury = pendingTreasury.newAddress;
    delete pendingTreasury;
}
```

**Benefits of Fix:**

* Ensures that collected fees remain within the protocol’s control and are not stranded in an old treasury.
* Enhances security by ensuring that the new treasury starts with the correct funds.

## Risk Assessment:

Severity: High

Reasoning:
If the fees collected by the treasury aren't properly transferred to the new address before updating it (that is always the case when calling `applyTreasuryUpdate()`), there is a risk that those funds could be lost. This is a significant issue, especially if the treasury holds substantial amounts of funds. The loss of fees would directly impact the contract’s ability to distribute or use those funds, which could be catastrophic for the contract’s intended functionality.


## [H-5] Lack of enforcement of the `MAX_TOTAL_LOCKED_AMOUNT`

### Summary

The issue allows users to lock more tokens than intended when creating the lock, leading to potential exploitation of the boost calculation mechanism and potential imbalances in the veRAAC ecosystem.

### Vulnerability Details

**Issue:**  `MAX_TOTAL_LOCKED_AMOUNT` Not Enforced

The `veRAACToken` contract defines a maximum total locked amount (`MAX_TOTAL_LOCKED_AMOUNT`), intended to limit the total amount of RAAC that can be locked in the system. However, the contract does *not* enforce this limit.  While individual lock amounts (`MAX_LOCK_AMOUNT`) and lock durations (`MAX_LOCK_DURATION`) are checked, there is no check to prevent the *sum* of all locked amounts from exceeding `MAX_TOTAL_LOCKED_AMOUNT`.

**Location:** `veRAACToken.sol:lock`

In `veRAACToken.sol` the `lock` function can be called.

Inside, the folowing functions are being called:

```solidity
_lockState.createLock(msg.sender, amount, duration);
```

where this state is updated `state.totalLocked += amount;`

```solidity
_updateBoostState(msg.sender, amount);
```

where `_boostState.totalWeight = _lockState.totalLocked;` is updated and again `totalLocked` is not checked if it exieds the `MAX_TOTAL_LOCKED_AMOUNT`

```solidity
        // Mint veTokens
        _mint(msg.sender, newPower);
```

where `veTokens` are minted which can cause a inflation of those tokens.

### Example:

1. `MAX_TOTAL_LOCKED_AMOUNT` is set to 1,000,000,000e18

2. The curent locedAmount = 990,000,000e18

3. Alice locks 10,000,000e18 RAAC.

4. Bob locks 10,000,000e18 RAAC.

5. Ema locks 10,000,000e18 RAAC.

The contract allows Bob's and Ema's lock, even though the total locked amount (1,020,000,000e18) now significantly exceeds `MAX_TOTAL_LOCKED_AMOUNT`.

### Impact

**Severity:** High

**Potential Consequences:**

* **Unfair Rewards:**  The boost calculation mechanism in `veRAACToken` relies on the total locked amount. Inflating the total locked amount.

* **Potential for Manipulation:** The boost mechanism is tied to governance, the inflated boost could allow manipulation of governance proposals.

### Tools Used

* **Manual Code Review**

### Recommendations

1. **Enforce** **`MAX_TOTAL_LOCKED_AMOUNT`:** Add check in the `lock` function to ensure that the total locked amount does not exceed `MAX_TOTAL_LOCKED_AMOUNT`.  The provided code snippet below demonstrates the necessary changes:

```Solidity
    function lock(uint256 amount, uint256 duration) external nonReentrant whenNotPaused {
        // ... (other checks)

        if (_lockState.totalLocked + amount > _lockState.maxTotalLocked) {
            revert TotalLockedAmountExceeded(); // Custom error
        }

        // ... (rest of the lock logic)
    }
```

(Recommended code is not tested)


## [H-6] Funds get stuck in the `Treasury`             



## Summary

This is due to the fact that instead of calling the `Treasury::deposit` function, the `FeeCollector` just transfers the RAACToken to the treasury

## Vulnerability Details

The problem can be located in the `FeeCollector::_processDistributions` function:

```solidity
   function _processDistributions(
        uint256 totalFees,
        uint256[4] memory shares
    ) internal {
        uint256 contractBalance = raacToken.balanceOf(address(this));
        if (contractBalance < totalFees) revert InsufficientBalance();

        if (shares[0] > 0) {
            uint256 totalVeRAACSupply = veRAACToken.getTotalVotingPower();
            if (totalVeRAACSupply > 0) {
                TimeWeightedAverage.createPeriod(
                    distributionPeriod,
                    block.timestamp + 1,
                    7 days,
                    shares[0],
                    totalVeRAACSupply
                );
                totalDistributed += shares[0];
            } else {
                shares[3] += shares[0]; // Add to treasury if no veRAAC holders
            }
        }

        if (shares[1] > 0) raacToken.burn(shares[1]);
        if (shares[2] > 0) raacToken.safeTransfer(repairFund, shares[2]);

@>        if (shares[3] > 0) raacToken.safeTransfer(treasury, shares[3]);
    }
```

In the pretty last line of the function we see that the `RAAC` tokens are just transferred to the treasury instead of deposited into it. This will lead to funds locked in the treasury since when the funds are transferred like this, the `Treasury::_totalValue` variable and `Treasury::_balances` mapping won't be updated

## Impact

`RAAC` token will be forever locked in the treasury

## Tools Used

Manual Review

## Recommendations

call the `Treasury::deposit` function instead of just transferring the tokens        


## [M-1] Unsafe ERC20 Transfers: Vulnerability in Deposit and Withdraw Functions

## Summary

The codebase contains a vulnerability in the deposit and withdraw functions related to `ERC20` token transfers. The implementation directly calls transferFrom and transfer without using `safeTransferFrom` or `safeTransfer`, which can lead to inconsistencies if the `ERC20` token returns false instead of reverting on failure. This could result in incorrect updates to `_balances` and `_totalValue`, leading to potential loss of funds or incorrect accounting within the contract.

## Vulnerability Details

The key issue lies in the way ERC20 token transfers are handled:

```solidity
    function deposit(address token, uint256 amount) external override nonReentrant {
        if (token == address(0)) revert InvalidAddress();
        if (amount == 0) revert InvalidAmount();
        
        //@audit use safeTransferFrom
        IERC20(token).transferFrom(msg.sender, address(this), amount);
        _balances[token] += amount;
        _totalValue += amount;
        
        emit Deposited(token, amount);
    }
```

```solidity
    function withdraw(
        address token,
        uint256 amount,
        address recipient
    ) external override nonReentrant onlyRole(MANAGER_ROLE) {
        if (token == address(0)) revert InvalidAddress();
        if (recipient == address(0)) revert InvalidRecipient();
        if (_balances[token] < amount) revert InsufficientBalance();
        
        _balances[token] -= amount;
        _totalValue -= amount;
        //@audit use safeTransfer
        IERC20(token).transfer(recipient, amount);
        
        emit Withdrawn(token, amount, recipient);
    }
```

Many ERC20 tokens adhere to the OpenZeppelin standard, where `transferFrom` and `transfer` revert on failure. However, some tokens return false instead of reverting. The current implementation does not check for a return value, which means that if a token fails to transfer, `_balances` and `_totalValue` will still be updated incorrectly. This could create an accounting discrepancy and potential loss of funds.

## Impact

* If a token returns false on failure instead of reverting, `_balances` and `_totalValue` will be updated despite the transfer not actually succeeding.

* This could lead to incorrect balances being reflected in the contract, causing overstatements of funds held in the treasury.

* A malicious actor could exploit this by depositing a token that does not revert on failure and then withdrawing more than they should be entitled to.

* Any token relying on false return (standard ERC20) values instead of reverts could cause silent failures, leading to unexpected behavior.

## PoC

You can add this test function to your test file located in 2025-02-raac/test/unit/core/collectors/Treasury.test.js

```solidity
    describe("SafeERC20 Transfers", () => {
        let failingToken;

        beforeEach(async () => {
            // Deploy a mock token that returns false instead of reverting
            const FailingToken = await ethers.getContractFactory("FailingToken");
            failingToken = await FailingToken.deploy("Failing Token", "FAIL", 18);
            await failingToken.mint(user1.address, ethers.parseEther("1000"));
        });

        it("should revert when deposit fails due to token returning false", async () => {
            await failingToken.connect(user1).approve(treasury.getAddress(), ethers.parseEther("100"));
            await expect(
                treasury.connect(user1).deposit(failingToken.getAddress(), ethers.parseEther("100"))
            ).to.be.revertedWith("SafeERC20: ERC20 operation did not succeed");
        });

        it("should revert when withdrawal fails due to token returning false", async () => {
            // Approve and deposit valid token to ensure there are funds in the contract
            await token.connect(user1).approve(treasury.getAddress(), ethers.parseEther("100"));
            await treasury.connect(user1).deposit(token.getAddress(), ethers.parseEther("100"));

            // Replace the token in contract balance mapping (mocking a scenario where it holds a failing token)
            await ethers.provider.send("hardhat_setStorageAt", [
                treasury.getAddress(),
                ethers.utils.keccak256(ethers.utils.defaultAbiCoder.encode(["address"], [failingToken.getAddress()])),
                ethers.utils.hexZeroPad(ethers.utils.hexlify(ethers.parseEther("100")), 32)
            ]);

            await expect(
                treasury.connect(manager).withdraw(failingToken.getAddress(), ethers.parseEther("50"), user2.address)
            ).to.be.revertedWith("SafeERC20: ERC20 operation did not succeed");
        });
    });
```

## Tools Used

Manual review.

## Recommendations

To prevent this vulnerability, use OpenZeppelin’s SafeERC20 library, which ensures that failed transfers revert instead of returning false. The correct implementation should be:

```solidity
import { SafeERC20 } from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

using SafeERC20 for IERC20;

IERC20(token).safeTransferFrom(msg.sender, address(this), amount);

IERC20(token).safeTransfer(recipient, amount);
```

This approach guarantees that the contract correctly handles token transfer failures, preventing incorrect balance updates and potential fund losses.



## [M-2] Fee-On-Transfer Incompatibility with Balance Checks

## Summary

The RAAC token contract implements a fee-on-transfer mechanism that deducts a portion of tokens for taxation (swap and burn). However, this introduces issues in functions that assume exact token transfers, particularly in `StabilityPool:depositRAACFromPool`. The function checks for a precise balance update after `safeTransferFrom`, which fails due to the fee deduction. This could prevent deposits or lead to unintended contract behavior.

## Vulnerability Details

The `depositRAACFromPool` function attempts to verify an exact token transfer by comparing pre- and post-transfer balances:

Location:  2025-02-raac\contracts\core\pools\StabilityPool\StabilityPool.sol \[326-337]

```solidity
uint256 preBalance = raacToken.balanceOf(address(this));
raacToken.safeTransferFrom(msg.sender, address(this), amount);
uint256 postBalance = raacToken.balanceOf(address(this));
if (postBalance != preBalance + amount) revert InvalidTransfer();
```

However, since RAAC imposes a swap and burn tax, the contract receives **less than** the expected `amount`. As a result, the balance check fails, reverting the transaction.

### **Tax Calculation in RAAC Transfers**

RAAC deducts a fee on transfer:

Location:  2025-02-raac\contracts\core\tokens\RAACToken.sol \[185-204]

```solidity
uint256 totalTax = amount.percentMul(baseTax);
uint256 burnAmount = totalTax * burnTaxRate / baseTax;
super._update(from, feeCollector, totalTax - burnAmount);
super._update(from, address(0), burnAmount);
super._update(from, to, amount - totalTax);
```

This means that `amount` sent by the sender is greater than what the contract actually receives, breaking deposit logic.

## Impact

* **Deposit Function Fails**: Users cannot deposit RAAC tokens from the liquidity pool due to failed balance checks.

## Tools Used

**Manual Review**

## Recommendations

**Option 1: Modify the deposit function to account for fee deductions**

**Option 2: Whitelist Contract to Avoid Tax**

Allow the deposit contract to be whitelisted, preventing fee deductions:

```solidity
raacToken.manageWhitelist(address(this), true);
```

## Conclusion

The fee-on-transfer mechanism in RAAC introduces security and usability concerns when interacting with contracts that rely on precise balance calculations. Implementing one of the above solutions can mitigate these issues and ensure smooth functionality.


## [M-3] Boost Multiplier Calculation Issue

### Summary

The `getBoostMultiplier` function in the contract contains a issue where the boost multiplier calculation always returns `MAX_BOOST` if `userBoost.amount > 0`, leading to an incorrect boost determination. This makes it unfair for users that heve much boost amount and ones that have just "1".

### Vulnerability Details

* **Affected Function:** `BoostController:getBoostMultiplier`
* **Issue:**
  ```solidity
  uint256 baseAmount = userBoost.amount * 10000 / MAX_BOOST;
  return userBoost.amount * 10000 / baseAmount;
  ```
  This logic results in a division where `baseAmount` cancels out `userBoost.amount * 10000`, always returning `MAX_BOOST` when `userBoost.amount > 0`.

### Impact

* **Loss of Intended Functionality:** The function does not differentiate boost levels correctly.

## Example of the current implementation:

* Alice (userBoost.amount = 0) → Returns MIN\_BOOST (as per function logic)

* Bob (userBoost.amount = 1) → Returns MAX\_BOOST

* Charlie (userBoost.amount = 10000) → Returns MAX\_BOOST

### Recommendations

* Correct the calculation to reflect the intended scaling
* Ensure `userBoost.amount` is factored in a way that prevents constant max boost returns.

### Conclusion

The current implementation of `getBoostMultiplier` results in all users receiving `MAX_BOOST`, rendering the function ineffective. Fixing the calculation will restore accurate boost determination and prevent potential manipulation.


## [I-1] Redundant code

## Summary

Severity: Informational

If check is redundant, just remove it.

Both `if` and `else` blocks are the same.

Location: 2025-02-raac\contracts\core\governance\gauges\BaseGauge.sol        Line\[185-210]

```solidity
    /**
     * @notice Updates weights for time-weighted average calculation
     * @dev Creates new period or updates existing one with new weight
     * @param newWeight New weight value to record
     */
    function _updateWeights(uint256 newWeight) internal {
        uint256 currentTime = block.timestamp;
        uint256 duration = getPeriodDuration();
        
        if (weightPeriod.startTime == 0) {
            // For initial period, start from next period boundary
            uint256 nextPeriodStart = ((currentTime / duration) + 1) * duration;
            TimeWeightedAverage.createPeriod(
                weightPeriod,
                nextPeriodStart,
                duration,
                newWeight,
                WEIGHT_PRECISION
            );
        } else {
            // For subsequent periods, ensure we're creating a future period
            uint256 nextPeriodStart = ((currentTime / duration) + 1) * duration;
            TimeWeightedAverage.createPeriod(
                weightPeriod,
                nextPeriodStart,
                duration,
                newWeight,
                WEIGHT_PRECISION
            );
        }
    }
```
