# ERC20 Transfer Compatibility Issue

### Severity: Medium

### Description:
The protocol intends to support all ERC20 tokens, but the implementation uses the original transfer functions. Some tokens (like USDT) do not implement the EIP20 standard correctly, and their `transfer`/`transferFrom` functions return `void` instead of a success boolean. Calling these functions with the correct EIP20 function signatures will revert.

Problematic lines: 69 and 84
```solidity
Locking.sol
47:  function lock(
48:       address account,
49:       address _delegate,
50:       uint96 amount,
51:       uint32 slopePeriod,
52:       uint32 cliff
53:       ) external override returns (uint256) {
54:       require(amount >= 1e18, "amount is less than minimum");
55:       require(cliff <= MAX_CLIFF_PERIOD, "cliff too big");
56:       require(slopePeriod <= MAX_SLOPE_PERIOD, "period too big");
57:       require(account != address(0), "account is zero");
58:       require(_delegate != address(0), "delegate is zero");
59:
60:       counter++;
61:
62:       uint32 currentBlock = getBlockNumber();
63:       uint32 time = getWeekNumber(currentBlock);
64:       addLines(account, _delegate, amount, slopePeriod, cliff, time, currentBlock);
65:       accounts[account].amount = accounts[account].amount + (amount);
66:
67:       // slither-disable-next-line reentrancy-events
68:       //@audit require will always revert if any wierd ERC20 is used
69:       require(token.transferFrom(msg.sender, address(this), amount), "transfer failed");
70:
71:       emit LockCreate(counter, account, _delegate, time, amount, slopePeriod, cliff);
72:       return counter;
73:  }
```
Users cannot lock funds using tokens like USDT, effectively excluding them from participating in the protocol's functionality.

```solidity
Locking.sol
78:  function withdraw() external {
79:       uint96 value = getAvailableForWithdraw(msg.sender);
80:       if (value > 0) {
81:            accounts[msg.sender].amount = accounts[msg.sender].amount - (value);
82:            // slither-disable-next-line reentrancy-events
83:            //@audit require will always revert if any wierd ERC20 is used
84:            require(token.transfer(msg.sender, value), "transfer failed");
85:       }
86:       emit Withdraw(msg.sender, value);
87:  }
```
A revert in this function means users' funds remain inaccessible (unless the user replaces the unusable token with another ERC20), creating a severe usability issue and potential reputational risk for the protocol.

### Impact:
Tokens that do not correctly implement the EIP20 standard (like USDT) will be unusable in the protocol, as they revert the transaction because of the missing return value.

### Recommended Mitigation:
We recommend using OpenZeppelin's SafeERC20 versions with the `safeTransfer` and `safeTransferFrom` functions. These handle the return value check as well as non-standard-compliant tokens.

```solidity
Locking.sol

69:  - require(token.transferFrom(msg.sender, address(this), amount), "transfer failed");
69:  + require(token.safeTransferFrom(msg.sender, address(this), amount), "transfer failed");

84:  - require(token.transfer(msg.sender, value), "transfer failed");
84:  + require(token.safeTransfer(msg.sender, value), "transfer failed");
```
---
