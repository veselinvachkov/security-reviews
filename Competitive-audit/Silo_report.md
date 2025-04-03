**Title: Silo Finance**

Auditor: **vesko210**

## Issues found
|Severtity|Number of issues found
| ------- | -------------------- |
| Medium  | 2                    |

# Findings

## [M-1]Irrevocable claimer control, poses security risk for users

### Finding description:
The current implementation of the `BaseIncentivesController` contract restricts the ability to set a claimer to only the contract owner. This limitation introduces a potential issue where users do not have control over who can claim rewards on their behalf. Additionally, there is no functionality to revoke or change a claimer once it has been assigned.

### Impact:
1. Irrevocable Claimer Risk: Once a claimer is assigned, they permanently hold the ability to claim rewards on behalf of the user. This creates a risk where a malicious or compromised claimer could claim rewards indefinitely without a way of solving the problem. In a scenario where a claimer is compromised (which the owner has no way to insure will not happend) action can not be taken to defend a user or the contract from lossing his rewards. In some cases, having a claimer can be necessary, especially if there is a contract that does not have a native claim function or supports delegation, but having no way to revoke a claimer, would mean that a compromised claimer could permanently exploit the protocol which, leading to unauthorized claims, potential fund loss, and a severe security risk with no recourse for recovery.
2. Loss of User Control: Since only the owner can set a claimer, users have no autonomy over this crucial security function. If a claimer is set without the user's consent or against their interest, they cannot change it, nor can the owner.
### Proof of Concept:
```solidity
Claimer Assignment: The function setClaimer() is restricted to the contract owner:
  function setClaimer(address _user, address _caller) external virtual onlyOwner {
      require(_user != address(0), ZeroAddress());
      require(_caller != address(0), ZeroAddress());
      
      _authorizedClaimers[_user] = _caller;
      emit ClaimerSet(_user, _caller);
  }
```
Issue: There is no mechanism to remove an existing claimer, allowing indefinite control over claims.

### Recommended mitigation steps
Allow Users to Set or Update Their Claimer
```solidity
function setClaimer(address _claimer) external {
    require(_claimer != address(0), "Claimer cannot be zero address");
    _authorizedClaimers[msg.sender] = _claimer;
    emit ClaimerSet(msg.sender, _claimer);
}
```
Introduce a function to allow users to remove their claimer
```solidity
function revokeClaimer() external {
    delete _authorizedClaimers[msg.sender];
    emit ClaimerRevoked(msg.sender);
}
```
### Severity explained:
This issue is a medium severity because, while the owner is trusted, the inability for users to manage their own claimers introduces unnecessary risk. Claimers can set, but once set, the user has no control over revoking access to thair rewards. Also at one point it might be needed for a claimer to be set but in the future it might cause a treat and be a issue which can not be solved.


## [M-2]Denial of service and exceptionally large fees in claimRewards() via gas exhaustion from large logic array

### Finding description:
The `claimRewards()` function in the SiloVault contract is vulnerable to a Denial of Service. This function relies on the `INCENTIVES_MODULE.getAllIncentivesClaimingLogics()` call to fetch an array of incentive logic contract addresses. The array is then iterated over, and the `claimRewardsAndDistribute()` function of each logic contract is invoked using delegatecall.
```solidity
    function _claimRewards() internal virtual {
@>      address[] memory logics = INCENTIVES_MODULE.getAllIncentivesClaimingLogics();
        bytes memory data = abi.encodeWithSelector(IIncentivesClaimingLogic.claimRewardsAndDistribute.selector);

        for (uint256 i; i < logics.length; i++) {
@>          (bool success, ) = logics[i].delegatecall(data);
            if (!success) revert ErrorsLib.ClaimRewardsFailed();
        }
    }
```
The `getAllIncentivesClaimingLogics()` function will copy the entire storage to memory, which can be quite expensive. This is designed to mostly be used by view accessors that are queried without any gas fees. The function copies the entire storage of market/logic addresses into memory. If the size of the array grows large enough, the gas cost for copying this array, combined with the gas consumption of the subsequent delegatecall operations, could exceed the block gas limit. This would result in a DoS, blocking all rewards from being claimed and distributed.

### Impact:
The vulnerability could make the `claimRewards()` function unusable if the cumulative size of the logic contracts exceeds a certain point. As the gas cost for each execution grows, it can reach a point where the function cannot complete within the block gas limit.
Also, even if the block gas limit is not exceeded, the function will produce super high fees. The approximate gas cost will be 26,000 gas per call to `(bool success, ) = logics[i].delegatecall(data)`. Having only 100 logics will generate an estimate of 2_600_000 gas.

### Scenario:
The workflow of the `claimRewards()` function is as follows:

1. `claimRewards()` function gets called on the SiloVault contract.

2. Next, `_claimRewards()` is invoked. Inside this function, `getAllIncentivesClaimingLogics()` is called on the IncentivesModule to fetch an array of logic contract addresses (logics[]).

3. This operation will copy the entire storage to memory, which is going to be quite expensive:
```solidity
    //@audit it is used in a not view function, which leads to DOS
    function getAllIncentivesClaimingLogics() external view virtual returns (address[] memory logics){
    /*
     * WARNING: This operation will copy the entire storage to memory, which can be quite expensive. This is designed
     * to mostly be used by view accessors that are queried without any gas fees. Developers should keep in mind that
     * this function has an unbounded cost, and using it as part of a state-changing function may render the function
     * uncallable if the set grows to a point where copying to memory consumes too much gas to fit in a block.
     */
        //@audit !!! gets all markets !!!
@>      address[] memory markets = _markets.values();

        //@audit address[] memory logics gets filled with unlimited amount of logics
@>      logics = _getAllIncentivesClaimingLogics(markets);
    }

    function _getAllIncentivesClaimingLogics(
        address[] memory _marketsInput
    ) internal view virtual returns (address[] memory logics) {
        uint256 totalLogics;

        for (uint256 i = 0; i < _marketsInput.length; i++) {
            unchecked {
                // safe to uncheck as we will never have more than 2^256 logics
@>              totalLogics += _claimingLogics[IERC4626(_marketsInput[i])].length();
            }
        }

        logics = new address[](totalLogics);

        uint256 index;
        for (uint256 i = 0; i < _marketsInput.length; i++) {
            address[] memory marketLogics = _claimingLogics[IERC4626(_marketsInput[i])].values();

            for (uint256 j = 0; j < marketLogics.length; j++) {
                unchecked {
                    // safe to uncheck as we will never have more than 2^256 logics
@>                  logics[index++] = marketLogics[j];
                }
            }
        }
    }
}
```
4. Once `getAllIncentivesClaimingLogics()` returns the `logics[]` array, a delegatecall is made to invoke `claimRewardsAndDistribute()` on each logic contract.

5. If any delegatecall fails, the entire `claimRewards()` function reverts.

### Recommended mitigation steps
Limit the Number of Claiming Logic Contracts:

- Implement a cap on the number of logic contracts each market can hold. This will ensure that the `getAllIncentivesClaimingLogics()` function remains efficient and safe to use, preventing gas limit issues that could lead to DoS attacks.
- Introduce a mechanism to limit the number of claiming logic contracts that can be added.