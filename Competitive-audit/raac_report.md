## Title Unsafe ERC20 Transfers: Vulnerability in Deposit and Withdraw Functions

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
