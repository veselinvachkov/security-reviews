## [H-1] Unrestricted Boost Delegation

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


## [H-2] Funds Locked Due to Misconfigured Treasury Address

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
