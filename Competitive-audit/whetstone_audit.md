### Severity: Low

### Description: 
The `lockPool` function in `DERC20.sol` not only locks the pool but also updates the pool address.This design forces the owner to reassign the pool address each time the pool is locked,increasing the risk of accidentally setting the pool to an incorrect address.Additionally, the `pool` and `isPoolUnlocked` variables are not inherently linked in a mapping ,meaning the pool lock functionality does not require the pool address to change. Removing the pool address update from this function would enhance safety and save gas by avoiding unnecessary reads/writes to the state variable `pool`.


```solidity
DERC20.sol
134:    function lockPool(
135:        address pool_
136:    )external onlyOwner {
137:        pool = pool_;
138:        isPoolUnlocked = false;
139:    }
```


### Impact:

* Accidental Misconfiguration: Increases the risk of assigning an incorrect or unintended address to the pool during the lock operation.
* Gas Inefficiency: Modifying the pool address unnecessarily consumes gas for additional reads and writes, especially when the update is not always required.
* Operational Complexity: Owners must track and reassign the correct pool address whenever locking the pool, leading to potential errors during frequent operations.



### Recommended Mitigation:

* Decouple the pool locking functionality from the pool address update.
* Introduce a separate function to update the pool address if required, leaving lockPool solely responsible for toggling the lock state.

```solidity
function updatePoolAddress(
        address pool_
    ) external onlyOwner {
        pool = pool_;
    }

function lockPool() external onlyOwner {
    isPoolUnlocked = false;
}

```

### Severity explained: 
* Lack of Safeguards: The function currently does not enforce validation of the new pool address, leaving room for unintentional or even malicious misconfigurations. 

* Loss of Funds or Operational Disruption: If the pool address is set to an unintended address (a zero address, an incorrect contract, or an external wallet), funds routed to the pool could be lost or mismanaged. 
This directly impacts the contract's functionality and the stakeholders reliant on it.

* Gas Cost Inefficiency: Repeated unnecessary writes to the state variable pool result in wasted gas, which accumulates over time, increasing costs for the contract owner or users.


---