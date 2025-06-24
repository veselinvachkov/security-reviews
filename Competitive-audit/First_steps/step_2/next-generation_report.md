# M1: Missing Blacklist Check in `Burn` Function

**Impact**: The `burn` function does not check if the caller is blacklisted, allowing blacklisted minters to burn tokens. This bypasses security restrictions, which could lead to misuse, especially if blacklisting is meant to restrict such actions. 
The blacklisted Minter still has his Role because curing the blacklisting proces it is not taken from them, also the `burn` function does not check if the Minter is blacklisted but only if he is `MASTER_MINTER` or `MINTER_ROLE`

## Proof of Concept:
The `burn` function currently does not verify if `msg.sender` is blacklisted before allowing the burn:

```solidity
/**
* @dev allows a minter to burn some of its own tokens
* Validates that caller is a minter and that sender is not blacklisted
* amount is less than or equal to the minter's account balance
* @param amount uint256 the amount of tokens to be burned
*/
function burn(uint256 amount) public virtual {
    if (!hasRole(MASTER_MINTER, _msgSender()) && !hasRole(MINTER_ROLE, _msgSender()))
        revert NotMinter(_msgSender());
    if (!_operating) revert OperationsOff();
        _burn(_msgSender(), amount);
        emit Burn(_msgSender(), amount);
}
```
## Recommended Mitigation Steps:
Add Blacklist Check: Ensure the caller is not blacklisted before burning:

**Severity: Medium** 
(Likelihood: Medium as all blacklisted minters will still be able to burn their tokens, Impact: Medium as the tokens are being burned and not minted)

**Links to affected code**
[burn function] https://github.com/code-423n4/2025-01-next-generation/blob/499cfa50a56126c0c3c6caa30808d79f82d31e34/contracts/ERC20ControlerMinterUpgradeable.sol#L169-#L175


# M2: User Funds Could Be Lost in Transaction Fee Due to Unrestricted _txfeeRate

## Finding description and impact
**Impact**: The current implementation of the `calculateTxFee` function in the `FeesHandlerUpgradeable` contract allows the transaction fee rate (`_txfeeRate`) to be set to a maximum value equal to `FEE_RATIO`, which is 100%. This means that, in theory, the transaction fee could be set to 100% of the transaction amount. If the fee rate is set to such an extreme value, the contract would take all the tokens from a transaction as the fee, effectively leaving the sender with nothing. 

This scenario could occur unintentionally or maliciously, and it presents a significant risk to users interacting with the contract. If the transaction fee rate is manipulated to this extreme, it could lead to a total loss of funds for users.

## Proof of Concept

The function `calculateTxFee` in the `FeesHandlerUpgradeable` contract performs the calculation as follows:

```solidity
function calculateTxFee(uint256 txAmount) public view returns (uint256) {
    return (txAmount * _txfeeRate) / FEE_RATIO;
}
```

The `_txfeeRate` is set via the `setTxFeeRate` function, which checks that the fee rate does not exceed `FEE_RATIO` or go below zero:

```solidity
function setTxFeeRate(uint256 newTxFeeRate) public virtual {
    if (newTxFeeRate > FEE_RATIO || newTxFeeRate < 0) revert InvalidFeeRate(FEE_RATIO, newTxFeeRate);
    _txfeeRate = newTxFeeRate;
    emit TxFeeRateUpdated(newTxFeeRate);
}
```

However, the fee rate can still be set to 100% of the `FEE_RATIO` (`_txfeeRate == 10000`), which means the entire transaction amount could be consumed as a fee.

## Recommended mitigation steps

Modify the `setTxFeeRate` function to restrict the `_txfeeRate ` from being set too high.

**Links to affected code**
[_txfeeRate] https://github.com/code-423n4/2025-01-next-generation/blob/499cfa50a56126c0c3c6caa30808d79f82d31e34/contracts/FeesHandlerUpgradeable.sol#L52-#L54
[calculateTxFee] https://github.com/code-423n4/2025-01-next-generation/blob/499cfa50a56126c0c3c6caa30808d79f82d31e34/contracts/FeesHandlerUpgradeable.sol#L28-#L32

**The likelihood is low, but the potential impact is high, so the severity should be rated as medium.**
