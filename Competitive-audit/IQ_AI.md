## Findings and Recommendations

### 1. Violation of Checks-Effects-Interactions Pattern
(The severity is low because there is not really a direct problem, but the convention must be followed)

Link to afected code: 
https://github.com/code-423n4/2025-01-iq-ai/blob/main/src/BootstrapPool.sol#L195-#L201

### Issue:  
The function executes external calls before updating the contract’s state variables, making it vulnerable to potential reentrancy attacks if future changes are made to the code.

### Recommendation:  
Follow the **Checks-Effects-Interactions** pattern by updating state variables before performing external interactions:

```solidity
function _sweepFees() internal {
    address feeTo = LiquidityManager(owner).owner();
    
    uint256 currencyFees = currencyTokenFeeEarned;
    uint256 agentFees = agentTokenFeeEarned;

    // **Update state before making external calls**
    currencyTokenFeeEarned = 0;
    agentTokenFeeEarned = 0;

    currencyToken.safeTransfer(feeTo, currencyFees);
    agentToken.safeTransfer(feeTo, agentFees);
}
```

### 2. Uncapped Swap Limits
Link to afected code: 
https://github.com/code-423n4/2025-01-iq-ai/blob/main/src/BootstrapPool.sol#L180

## Issue:
The `maxSwapAmount` function in the `BootstrapPool` contract contains a vulnerability where the maximum amount of tokens that can be swapped exceeds the actual balance of the `agentToken` in the contract. Specifically, the function sets `_amountIn = type(uint256).max`, allowing an attacker to request the largest possible amount for a swap, which could lead to issues if the `agentToken` balance is insufficient.

This can cause problems like failed transactions, overflows, or potential exploits where the contract function is used. Also a user using this function will recive quite a strange number for `maxSwapAmoun`: (115792089237316195423570985008687907853269984665640564039457584007913129639935)


### Recommended Solution:

The `maxSwapAmount` function should call the `getAmountIn` function for both cases, ensuring consistent calculation of the swap amount based on available liquidity.

Here is the updated code:

```solidity
function maxSwapAmount(address _tokenIn) public view returns (uint256 _amountIn) {
    if (_tokenIn == address(currencyToken)) {
        // Use getAmountIn to calculate the max swap amount based on available agentToken balance
        _amountIn = getAmountIn(agentToken.balanceOf(address(this)) - agentTokenFeeEarned, address(agentToken));
    } else if (_tokenIn == address(agentToken)) {
        // Calculate the amount based on the currencyToken balance and fee
        _amountIn = getAmountIn(currencyToken.balanceOf(address(this)) - currencyTokenFeeEarned, address(currencyToken));
    }
}
```