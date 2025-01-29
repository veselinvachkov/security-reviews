### Severity: Medium

### Lack of Minimum Token Amounts During Liquidity Provision

## Impact
The `finalizeFundraising` function lacks enforcement of minimum token amounts when adding liquidity to the pool. Specifically, the `amount0Min` and `amount1Min` parameters are hardcoded to `0` in the `INonfungiblePositionManager.MintParams` struct. This omission means that liquidity can be added regardless of unfavorable exchange rates, leading to potential loss of funds due to front-running, price impact, or imprecise token allocation.

---

## Proof of Concept

```solidity
INonfungiblePositionManager.MintParams memory params = INonfungiblePositionManager.MintParams(
    token0,
    token1,
    TICKING_SPACE,
    initialTick,
    upperTick,
    amountToken0ForLP,
    amountToken1ForLP,
    0, // amount0Min
    0, // amount1Min
    address(this),
    block.timestamp,
    sqrtPriceX96
);
```

In the `finalizeFundraising` function, after token distribution but before pool creation, the `amount0Min` and `amount1Min` values should enforce a minimum acceptable amount to ensure that liquidity is not added at unfavorable prices.

---

## Issue Location

- **File:** /src/Daao.sol

---

## Risks of Hardcoding amount0Min and amount1Min to 0

Hardcoding these values introduces the following risks:

1. **Front-Running Attacks:**
   - An attacker could manipulate the price just before liquidity is added, resulting in a disproportionate allocation of tokens.
   - This leads to suboptimal liquidity provision and potential losses.

2. **Significant Slippage:**
   - During market volatility, the tokens deposited might be converted at an unfavorable rate, leading to losses.

---

## Example Scenario

Suppose the intended token allocation is 4000 tokens of `token0` and 1000 tokens of `token1`. Due to market fluctuations, an attacker manipulates the price, causing an unfair ratio. With no minimum enforced, the liquidity is added regardless, leading to an unfair distribution of assets.

---

## Recommended Mitigation Steps

### Introduce minimum token return amounts

### Code Example

```solidity
uint256 expectedAmount0 = getExpectedAmount0(amountToken1ForLP);
uint256 expectedAmount1 = getExpectedAmount1(amountToken0ForLP);

INonfungiblePositionManager.MintParams memory params = INonfungiblePositionManager.MintParams(
    token0,
    token1,
    TICKING_SPACE,
    initialTick,
    upperTick,
    amountToken0ForLP,
    amountToken1ForLP,
    expectedAmount0 * 95 / 100, // 5% slippage tolerance
    expectedAmount1 * 95 / 100, // 5% slippage tolerance
    address(this),
    block.timestamp,
    sqrtPriceX96
);
```

 - By implementing these changes, the fundraising process becomes more resilient to unfavorable market conditions and manipulative attacks.
Think about a method to let the User to chose the slippage parameters himself.

---

## Links to Affected Code

- [Daao.sol#L340-341](https://github.com/daaoai/daaoai_contracts/blob/main/src/Daao.sol#L340-#L341)

