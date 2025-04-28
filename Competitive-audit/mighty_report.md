**Title: Mighty**

Auditor: **vesko210**

## Issues found
|Severtity|Number of issues found
| ------- | -------------------- |
| High    | 2                    |
| Medium  | 3                    |
| Low     | 1                    |

# Findings

## [H-1] Almost no rewards can be accumulated

## Summary

The protocol uses a global `rewardData[rewardToken].lastUpdateTime` per reward token, which can be frequent updated in `StakingRewards:updateReward` modifier which is called before all staking functions. This can prevent users from accruing any rewards.

---

## Finding Description

In the `StakingRewards` contract, the `rewardData[rewardToken].lastUpdateTime` is stored globally for each reward token and is updated every time a user interacts with the contract via `stake`, `withdraw`, `claim`, or other functions wrapped in the `updateReward` modifier.

```solidity
    modifier updateReward(address user) {
        for (uint256 i; i < rewardTokens.length; i++) {
            address rewardToken = rewardTokens[i];
            rewardData[rewardToken].rewardPerTokenStored = rewardPerToken(rewardToken);
@>          rewardData[rewardToken].lastUpdateTime = Math.min(rewardData[rewardToken].endTime, block.timestamp);

            if (user != address(0)) {
                userRewardsClaimable[user][rewardToken] =
                    earned(user, rewardToken, rewardData[rewardToken].rewardPerTokenStored);
                userRewardPerTokenPaid[user][rewardToken] = rewardData[rewardToken].rewardPerTokenStored;
            }
        }
        _;
    }
```

This creates a vulnerability where any user interaction causes the global `lastUpdateTime` to be updated. Consequently, in the function `rewardPerToken()`:

```solidity
    function rewardPerToken(address rewardToken) public view returns (uint256) {
        if (block.timestamp <= rewardData[rewardToken].startTime) {
            // new rewards not start
            return rewardData[rewardToken].rewardPerTokenStored;
        }

@>      uint256 dt =
            Math.min(rewardData[rewardToken].endTime, block.timestamp) - (rewardData[rewardToken].lastUpdateTime);

        if (dt == 0 || totalStaked == 0) {
            return rewardData[rewardToken].rewardPerTokenStored;
        }

        return rewardData[rewardToken].rewardPerTokenStored
        //rewardRate = totalRewards / (endTime - startTime)
            + (rewardData[rewardToken].rewardRate * dt * 1e18) / totalStaked;
    }
```

The time difference `dt` can be kept close to zero by frequent transactions by higher user interest ot artificially. This drastically reduces the rewards accumulated for all users.

- **Even if `dt` is around 300 = 6 minutes (which is almost unthinkable in the normal flow in the protocol) users below 0.1% holding of the total stakeed will still not be able to accumulate rewards.**
- **Also the `userRewardsClaimable[user][rewardToken]` is only increased once the user that has staked interacts with the contract this way even if he has staked for the whole pereod from start to end he will be not aworded anything simply because somebody would have updated the `rewardData[rewardToken].lastUpdateTime` to be equal to `rewardData[rewardToken].endTime`.**

This creates a systemic unfairness, breaking assumptions of:
- **Reward fairness per stake-time**

In the worst case, automated users can deny stakeholders their rewards entirely by continuously resetting `lastUpdateTime`.

---

## Impact Explanation

**Impact: High**

If the staking functionality is largely used the reward calculation gets updated too frequantly which will reduce the users generated rewards close to zero.

This vulnerability also allows malicious users to manipulate global reward calculation, blocking or significantly reducing rewards for all other users. It breaks the core staking guarantee of earning proportional rewards over time and enables denial-of-rewards via a griefing vector.

While no funds are directly stolen, this allows indirect theft of staking time value and undermines the system's fairness.

---

## Likelihood Explanation

**Likelihood: High**

The issue is **highly likely** to be exploited in practice because:
- It requires only standard, low-cost user actions (`stake`, `withdraw`, `claim`), (`update`, `withdrawByLendingPool`, `setReward`)
- There is **no rate limit or protection** against rapid interactions or personal staking time for the calculated rewards.
- The vulnerability can be exploited **accidentally or intentionally**

Any user, contributes for the lowered rewards of the others, especially those with automated tools.

---

## Proof of Concept

1. User A stakes **10 token**.
2. User B stakes **5000 tokens**.
3. User A sets up a bot to call `stake(10)` and `withdraw(10)` repeatedly within short time intervals, effectively resetting `lastUpdateTime` on every interaction.
4. User B remains passive, expecting to accumulate rewards over time.
5. After some time, User B calls `claim()` and observes that their `userRewardsClaimable` has barely increased or remains near zero.
   
Because `dt` is kept artificially small by User A's constant interactions, the calculated reward increment in `rewardPerToken` is negligible. This causes the `earned` value for User B and every other user to truncate to 0 or be at most up to the 100 depending 
on how much as a percentage the user has staked agains `totalStaked` due to:

```solidity
    function rewardPerToken(address rewardToken) public view returns (uint256) {
        if (block.timestamp <= rewardData[rewardToken].startTime) {
            // new rewards not start
            return rewardData[rewardToken].rewardPerTokenStored;
        }

        uint256 dt =
            Math.min(rewardData[rewardToken].endTime, block.timestamp) - (rewardData[rewardToken].lastUpdateTime);

        if (dt == 0 || totalStaked == 0) {
            return rewardData[rewardToken].rewardPerTokenStored;
        }

        return rewardData[rewardToken].rewardPerTokenStored
        //rewardRate = totalRewards / (endTime - startTime)
            + (rewardData[rewardToken].rewardRate * dt * 1e18) / totalStaked;
    }
```

For users that have less than 1% of the total staked it is almost imposible to earn rewards.

## Calculations
Lets just say the contract is in this state: 
- totalStaked = `10_000_000`
- totalRewards = `1_000_000`
- endTime - startTime = `260_000`
- userStaked = `10_000 = 0,1% of the totalStaked`

**1 First calculate `rewardPerToken`:**
Lets say that the last update was 6 minutes ago
dt = `300`
rewardPerToken = `0 + (totalRewards / endTime - startTime) * dt * 1e18 / totalStaked`
rewardPerToken = `900 * 1e18 / 10_000_000`
Now the flow goes to `earned` which isgoing to be sumed with `userRewardsClaimable[user][rewardToken]`
earned = `(userStaked * rewardPerToken) / 1e18 + 0`
earned = `10_000 * 900 * 1e18 / 10_000_000 / 1e18`
earned = `0`

**2 Another scenario**
If the user had staked 1% of the total staked but the last update was 30 seconds ago dt = 30
The user is still going to get nothing.


## Recommendation

**Fix: Track rewards in a more isolated and fair manner.**

### Introduce per-user time tracking:

Track `lastUpdateTime` per user per reward token. This isolates users completely and this way a reasonable amount of rewards can be accumulated.



## [H-2] First Depositor Exploitation

## Summary
The protocol allows the first depositor to manipulate the pool's liquidity-to-eToken ratio by donating tokens after their initial deposit. This results in significant dilution of future deposits and enables the first depositor to redeem vastly more than they originally deposited, leading to critical financial loss for other users.

---

## Finding Description
The `reserveToETokenExchangeRate()` function sets an exchange rate of `1e18` (1:1) when either `totalLiquidity` or `totalETokens` is zero. This logic, intended for initializing the system, opens the door for manipulation if not properly safeguarded.

Because the actual token balance is used to compute the exchange rate in subsequent deposits and redemptions, a malicious actor can exploit this by making a minimal initial deposit (e.g., 1 token) and then *donating* a large amount of tokens directly to the contract (e.g., 1000 tokens), thereby inflating the liquidity without minting corresponding eTokens.

Future users depositing after this manipulation will receive no `eTokens` due to the diluted exchange rate. The attacker can then redeem their original 1 eToken for the entire pool balance, effectively stealing all liquidity.

---
 
## Scenario:
This inflation attack works **only for the first depositor**:

1 **Initial State**  
- `totalLiquidity = 0`, `totalETokens = 0`  
- Alice calls `deposit(1)` → `_deposit()`  
- Exchange rate: `1e18` (1:1 default for first deposit)  
- Alice receives: `1 eToken`  
**State:** `totalLiquidity = 1`, `totalETokens = 1`

2 **Donation**  
- Alice sends 1000 tokens **directly** to the eToken contract (not via `deposit()`)  
**State:** `totalLiquidity = 1001`, `totalETokens = 1`

3 **Bob Deposits**  
State: totalLiquidity = 2001, totalETokens = 1
- Bob deposits 1000 via `deposit()`  
- eTokenAmount = `1000 * (1 * 1e18 / 1001) / 1e18 = 1000 / 1001 = 0`  
- Bob receives 0 eTokens  
**State:** `totalLiquidity = 2001`, `totalETokens = 1`

4 **Alice Redeems**  
- Alice redeems 1 eToken  
- Exchange rate = `2001 * 1e18 / 1 = 2001e18`  
- Alice receives: `1 * 2001e18 / 1e18 = 2001`  
**Final State:** Alice gains 1000 profit, Bob loses everything

---
## Code sniped

```solidity
    function _deposit(uint256 reserveId, uint256 amount, address onBehalfOf) internal returns (uint256 eTokenAmount) {
        DataTypes.ReserveData storage reserve = getReserve(reserveId);
        require(!reserve.getFrozen(), Errors.VL_RESERVE_FROZEN);
        // update states
        reserve.updateState(getTreasury());

        // validate
        reserve.checkCapacity(amount);
 
@>      uint256 exchangeRate = reserve.reserveToETokenExchangeRate();

        // Transfer the user's reserve token to eToken contract
        pay(reserve.underlyingTokenAddress, _msgSender(), reserve.eTokenAddress, amount);

        // Mint eTokens for the user
@>      eTokenAmount = amount.mul(exchangeRate).div(Precision.FACTOR1E18);

        IExtraInterestBearingToken(reserve.eTokenAddress).mint(onBehalfOf, eTokenAmount);

        // update the interest rate after the deposit
        reserve.updateInterestRates();
    }

    /**
     * @dev Exchange Rate from reserve liquidity to eToken
     * @param reserve The Reserve Object
     * @return The Exchange Rate
     */
    function reserveToETokenExchangeRate(DataTypes.ReserveData storage reserve) internal view returns (uint256) {
@>      (uint256 totalLiquidity,) = totalLiquidityAndBorrows(reserve);
@>      uint256 totalETokens = IERC20(reserve.eTokenAddress).totalSupply();

@>      if (totalETokens == 0 || totalLiquidity == 0) {
            return Precision.FACTOR1E18;
        }
@>      return totalETokens.mul(Precision.FACTOR1E18).div(totalLiquidity);
    }
```

## Impact Explanation
**Impact: High**  
This bug enables an attacker to extract *all* of the protocol's liquidity at zero cost, creating a *complete loss of funds* for other users.

---

## Likelihood Explanation
**Likelihood: High**  
- The exploit is trivial to execute.
- It only requires being the first depositor.
- Donation via `transfer()` or Self Destruct is simple and does not rely on any privileged access.
- There is no mechanism preventing or reverting untracked liquidity additions.

This would likely be discovered by a malicious actor or bot immediately upon deployment.

---

## Recommendation
**Set Minimum Initial Liquidity**  
Add initial liquidity upon deployment, even a small initial deposit from the protocol to the reserve is going to be unexploitable and
unprofitable from any attaker.

---


## [M-1] Liquidation risk due to mutable liquidationDebtRatio

## Summary

Inside `ShadowRangeVault` there is a function `setLiquidationDebtRatio` that allows `liquidationDebtRatio` to retroactively invalidate previously safe positions. This can lead to unintended liquidation of user positions, even when the underlying assets remain healthy.

## Finding Description

The smart contract `ShadowRangeVault` includes a mechanism for tracking user leverage via a `debt ratio` and enforcing position safety with a configurable `liquidationDebtRatio` parameter. Users can only close or reduce their positions if their current debt ratio is **below** the `liquidationDebtRatio`.

The issue arises when an admin **lowers** the `liquidationDebtRatio` after users have already opened positions that were compliant under the prior configuration. The require check:

```solidity
require(getDebtRatio(positionId) < liquidationDebtRatio, "DRH");
```

is enforced on key user functions:

- `closePosition()`
- `reducePosition()`
- `claimRewards()`

This means that users whose positions were once valid may find themselves **blocked** from closing or reducing their positions due to a rule change — even if no action was taken on their part.

Once trapped, these positions become eligible for forced liquidation via `liquidatePosition()`.

## Impact Explanation

This impact is **High** severity due to:

- Force liquidations of previously valid positions without user fault.
- The user is going to be blocked from closing or reducing their position.
- Cause unexpected fund losses for the user with a opened position.

## Likelihood Explanation

The likelihood is **Low**:

- The likelihood of this happening is low because the function can only be called by the owner, and it's unlikely to be used frequently. However, the issue will occur every time the function is used and the liquidationDebtRatio is changed.
- **This problem doesn't require misconfiguration or malicious use, it will always happen whenever the function is triggered and the ratio is altered.**

## Proof of Concept

### Step-by-step Workflow

1. Vault is configured with:
   ```solidity
   liquidationDebtRatio = 8600; // 86%
   ```

2. User opens a position with:
   - Principal: $100
   - Borrow: $85 (debt ratio = 85%)

   This is valid under the current config.

3. Admin changes the ratio:
   ```solidity
   vault.setLiquidationDebtRatio(8000); // 80%
   ```

4. User attempts to close the position:
   ```solidity
   closePosition(positionId);
   // fails due to: require(getDebtRatio(positionId) < 8000)
   ```

5. Position is now **stuck** and can only be resolved via:
   ```solidity
   liquidatePosition(positionId);
   ```

   Resulting in a forced liquidation.

## Recommendation

### Make positions save the LiquidationDebtRatio when the position was opened

Store the `debtRatioAtOpen` per position when it is created, this completely mitigates the problem:

```solidity
position.liquidationThreshold = currentLiquidationDebtRatio;
```

And update the check to:

```solidity
require(getDebtRatio(positionId) < position.liquidationThreshold, "DRH");
```



## [M-2]Title: Insolvent position liquidation handling issue

## Summary
The protocol currently stops the liquidation if the debt exceeds the position value, even marginally. This can lead to *permanently locked* and *unrecoverable* protocol funds if a leveraged position becomes insolvent during price update delays combined with a volatile market.

## Finding Description
In the `ShadowRangeVault` contract, there is a liquidation check that require that after the liquidation there is 0 debt left:
```solidity
    (uint256 currentDebt0, uint256 currentDebt1) = getPositionDebt(positionId);
    require(currentDebt0 == 0 && currentDebt1 == 0, "Still debt");
```

However, when the position becomes insolvent (due to a sharp price drop + update lag), the code **does not allow liquidation**. Because the protocol deducts liquidation fees before attempting to repay the debt, the available collateral is further reduced. This can cause the position's remaining value to fall short of fully repaying the debt, triggering a failure in the liquidation process.
The protocol could still partially recover assets if the liquidation goes through, but instead **blocks liquidation entirely**, causing a potential permanent loss or locked funds for a long time.

## Impact Explanation
**Impact: High**

- The impact is going to affect the protocol as the debt that the user had borrowed from the protocol is going to be locked of lost.

## Likelihood Explanation
**Likelihood: Low**

- The likelihood is defenetly low as there must be a significant drop in the price and also a small lag in the price. But that is a possibility and should not be neglegated.

## Proof of Concept
**Scenario:**

1. User opens a position with 3x leverage position.
2. Sharp asset price crash occurs.
3. Price update is delayed or slightly lagging, just minutes can be enough in some cases.
4. Position becomes underwater (debt > position value) after crash.
5. When trying to liquidate:
    - The contract first takes the liquidation fee and the fee for the liquidator, which futrermore brings the position value down:
    ```solidity
    // handle liquidation fees
        if (token0Reduced > 0) {
            token0Fees = token0Reduced * vars.liquidationFee / 10000;
            pay(token0, address(this), vars.liquidationFeeRecipient, token0Fees);

            uint256 token0CallerFees = token0Fees * vars.liquidationCallerFee / 10000;
            if (token0CallerFees > 0) {
                pay(token0, address(this), caller, token0CallerFees);
            }

            token0Reduced = token0Reduced - token0Fees - token0CallerFees;
        }

        if (token1Reduced > 0) {
            token1Fees = token1Reduced * vars.liquidationFee / 10000;

            pay(token1, address(this), vars.liquidationFeeRecipient, token1Fees);

            uint256 token1CallerFees = token1Fees * vars.liquidationCallerFee / 10000;
            if (token1CallerFees > 0) {
                pay(token1, address(this), caller, token1CallerFees);
            }

            token1Reduced = token1Reduced - token1Fees - token1CallerFees;
        }
        ```
6. The users collateral tries to repay the debt but it is going to be incomplete.
7.  Funds become locked and potentionaly lost.


## Recommendation
**Fix:**  
- Liquidation fees can be taken after the debt gets payed, **even if there is no enough collateral to cover all the fees alow the liquidation to go through**, this is going to give the price a little (5% by default fee to the `liquidationFeeRecipient` and some more for the liquidator) more brething room before the position becomes insolvent.


Allow liquidation **even if** the position is insolvent, to recover *as much borrowed funds as possible*. 
This would avoid permanent locking of funds and improve the protocol’s resilience against volatility.

---

## [M-3] Missing slippage protection can result in users receiving less than expected

## Summary

The `reducePosition` function calls `_swapTokenExactInput` with `minAmountOut` set to zero. Additionally, within `_swapTokenExactInput`, `sqrtPriceLimitX96` is set to zero, which exposes users to high slippage and no price boundaries during the swap. This means there is no limit on price movement, allowing the swap to traverse through ticks and execute at potentially unfavorable prices.

## Issue Description

The `reducePosition` function allows users to reduce the value of their position, as shown in the following code:

```solidity
if (amount0ToSwap > 0) {
    _swapTokenExactInput(token0, token1, amount0ToSwap, 0);
}
if (amount1ToSwap > 0) {
    _swapTokenExactInput(token1, token0, amount1ToSwap, 0);
}
```

However, as seen in the code, there is no slippage protection mechanism in place. This is problematic because the token exchange rate is based on current market prices, which can change rapidly during periods of high volatility. 

While the transaction is pending in the mempool, the price may change significantly, resulting in a worse exchange rate when the transaction is finally executed.

The function:
 ```solidity
    function _swapTokenExactInput(address tokenIn, address tokenOut, uint256 amountIn, uint256 amountOutMinimum)
        internal
        returns (uint256 amountOut)
    {
        address router =
            IAddressRegistry(IVault(vault).addressProvider()).getAddress(AddressId.ADDRESS_ID_SHADOW_ROUTER);
        IERC20(tokenIn).approve(router, amountIn);

        amountOut = IShadowSwapRouter(router).exactInputSingle(
            IShadowSwapRouter.ExactInputSingleParams({
                tokenIn: tokenIn,
                tokenOut: tokenOut,
                tickSpacing: tickSpacing,
                recipient: address(this),
                deadline: block.timestamp,
                amountIn: amountIn,
                amountOutMinimum: amountOutMinimum,
                sqrtPriceLimitX96: 0
            })
        );
    }
```
"Swaps `amountIn` of one token for as much as possible of another token" but this does not guarantee that the user will receive the amount he desired.

**For example:**

1. A user sets `amount0ToSwap` and expects to receive 100 tokens1.
2. While the transaction is in the mempool, the price of the tokens changes.
3. Upon transaction execution, the user only receives 80 tokens1, resulting in a 20-token loss.
4. The user would not have agreed to such a trade and would have expected a minimum of 99 tokens1.

Since `minAmountOut = 0`, there is no safeguard to prevent the transaction from executing at this unfavorable price.

## Recommendation:

Allow users to specify a minimum amount they are willing to receive in order to prevent substantial losses from price slippage.

---


## [L-1]Fee-On-Transfer token exploit in staking mechanism

## Summary

The contract does not account for the possibility that the staking token may include a fee-on-transfer mechanism. This creates an inconsistency between the actual token balance of the contract and the recorded staking balances, enabling a draining attack through repeated staking and withdrawing.

---

## Finding Description

In the `StakingRewards` contract, the `stake()` function assumes the full `amount` passed to it is received by the contract:

```solidity
stakedToken.safeTransferFrom(msg.sender, address(this), amount);
balanceOf[onBehalfOf] += amount;
totalStaked += amount;
```

However, if the `stakedToken` includes a fee-on-transfer, then the contract receives fewer tokens than intended. Despite this, the user's staked balance and the global `totalStaked` are credited with the full `amount`.

This mismatch leads to an **overestimation of the contract's staking reserves**. Since the internal accounting does not reflect the actual token balance, a malicious user can exploit this to repeatedly stake and withdraw, draining the contract of all its assets.

The `withdraw()` function trusts the internal `balanceOf` accounting:

```solidity
require(stakedToken.transfer(to, amount), "transfer failed");
```

...without verifying that the contract actually received `amount` during the stake phase. This allows users to withdraw more than they contributed.

---

## Impact Explanation

**Impact: High**

This vulnerability allows an attacker to drain all staked funds from the contract over time. The internal accounting system assumes perfect token transfers, but if the token enforces a fee, the discrepancy compounds over repeated interactions.

Even honest users will unknowingly cause underfunding of the contract, but a malicious user can intentionally:
- Stake tokens (receiving full credit despite the transfer fee),
- Withdraw the full credited amount,
- Repeat the process: The user end up with the same value he started with but the contact loses all its fund, thay go towards the token with the fee.

**Over time, this can lead to loss of funds for all the stakers, which is explained how in the PoC section.**

---

## Likelihood Explanation

**Likelihood: low**

- No checks exist to validate that the full `amount` was received.

The required steps to exploit the issue (calling `stake` and `withdraw`) are normal user actions and cost very little gas.

---

## Proof of Concept

Assume a token that fees 5% on transfer:

1. Attacker stakes `1000` tokens.
   - Only `950` tokens are actually received by the contract.
   - `balanceOf[attacker]` and `totalStaked` are still increased by `1000`.

2. Attacker withdraws `1000` tokens.
   - The contract sends `1000` tokens back to the attacker.
   - The contract just lost `50` tokens that belonged to other users.

3. Repeat the cycle to drain the contract entirely.

This can be done even more aggressively with a bot that automates stake/withdraw every block.

---

## Recommendation

**Fix: Use actual received amount for accounting.**

Update the `stake()` function to measure the real amount received:

```solidity
function stake(uint256 amount, address onBehalfOf) external nonReentrant updateReward(onBehalfOf) {
    require(amount > 0, "amount = 0");

    uint256 balanceBefore = stakedToken.balanceOf(address(this));
    stakedToken.safeTransferFrom(msg.sender, address(this), amount);
    uint256 balanceAfter = stakedToken.balanceOf(address(this));

    uint256 actualReceived = balanceAfter - balanceBefore;

    balanceOf[onBehalfOf] += actualReceived;
    totalStaked += actualReceived;

    emit Staked(msg.sender, onBehalfOf, actualReceived);
}
```

Another option is to prevent usage with fee-on-transfer tokens

If you want to restrict all fee-on-transfer tokens, add an explicit check:

```solidity
require(actualReceived == amount, "fee-on-transfer tokens not supported");
```
