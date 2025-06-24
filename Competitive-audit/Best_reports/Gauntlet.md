**Title: Aere-V3**

Auditor: **vesko210**

## Issues found
|Severtity|Number of issues found
| ------- | -------------------- |
| High    | 0                    |
| Medium  | 2                    |
| Low     | 0                    |


# Findings

## [M-1] Solvers do not profit from solving directRedeem requests

## Summary

Solvers of **direct‐redeem** requests receive vault tokens but have no mechanism to convert them back into underlying without paying another solver tip + gas fees, threatening the liveness of direct redeem request processing.

## Finding Description

When a user calls `requestRedeem` with `isFixedPrice = true`, their vault units are escrowed in the Provisioner. A solver executing `_solveRedeemDirect` is paid those units.

However, **the solver now holds vault tokens** and **cannot exit** them back to the underlying asset (for example WETH), except by creating **another** async‐redeem request **and paying a new solver tip**. This breaks:

A rational solver will simply ignore direct‐redeem requests if they cannot immediately realize value, leaving user requests unfulfilled indefinitely. The only way a solver can profit from direct redeem request is if the user has made the request absolytely unprofitable for himselfe. **This breaks the functionality of an instant/direct redeem.**

## Impact Explanation

* **Liveness**: Direct redeem requests may never be solved if no solver can profitably convert the received vault tokens back.
* **Fair compensation**: Solvers risk ending at a net loss or at brack even after gas and tip fees, so they will refuse to solve direct‐redeems.

## Likelihood Explanation

Any solver bot will detect that they cannot realize the received units and simply omit those requests.

## Proof of Concept

This test shows how eve (the solver) solves 2 redeem direct requests, she that creates her own request to transfer her vault tokens for underlying which is the only way for direct profit for eve. Now we need somebody to execute the eve request and that is dan. After all this eve is left with slightly less undelying than in the start, this is because dan and charlie are paying just a little less the is eve, but thay can be made to be exactly 10% like in the eve request. In conclusion eve has got to pay around the same amount to realize her vault tokens, which with gas fees and time is not going to be worth it.

To run this test you need to import: import `"forge-std/console.sol";`
Add this function inside `Provisioner.t.sol` and run it with: `forge test --mt test_solveRequestsDirect_success_all_solvable -vvvv`

```solidity
function test_solveRequestsDirect_success_all_solvable() public {
        vm.prank(users.owner);
        provisioner.setTokenDetails(
            token,
            TokenDetails({
                syncDepositEnabled: false,
                asyncDepositEnabled: true,
                asyncRedeemEnabled: true,
                depositMultiplier: uint16(ONE_IN_BPS),
                redeemMultiplier: uint16(ONE_IN_BPS)
            })
        );

        Request memory redeem1 = _makeRequest({
            user: users.charlie,
            requestType: RequestType.DEPOSIT_AUTO_PRICE,
            tokens: 3000 ether,
            units: 3300 ether,
            solverTip: 0,
            deadline: 1 days,
            maxPriceAge: UNIT_PRICE_AGE
        });
        Request memory redeem2 = _makeRequest({
            user: users.dan,
            requestType: RequestType.REDEEM_FIXED_PRICE,
            tokens: 2000 ether,
            units: 2200 ether,
            solverTip: 0,
            deadline: 1 days,
            maxPriceAge: UNIT_PRICE_AGE
        });

        multiDepositorVault.mint(redeem1.user, redeem1.units);
        _approveToken(redeem1.user, IERC20(address(multiDepositorVault)), address(provisioner), redeem1.units);
        multiDepositorVault.mint(redeem2.user, redeem2.units);
        _approveToken(redeem2.user, IERC20(address(multiDepositorVault)), address(provisioner), redeem2.units);

        ERC20Mock(address(token)).mint(users.eve, redeem1.tokens + redeem2.tokens);

        uint256 eve_balanece_before_She_solves_in_underlying = token.balanceOf(users.eve);
        uint256 eve_balanece_before_She_solves_in_vault = IERC20(address(multiDepositorVault)).balanceOf(users.eve);

        // Make redeem requests
        vm.prank(redeem1.user);
        provisioner.requestRedeem(
            token, redeem1.units, redeem1.tokens, redeem1.solverTip, redeem1.deadline, redeem1.maxPriceAge, true
        );

        vm.prank(redeem2.user);
        provisioner.requestRedeem(
            token, redeem2.units, redeem2.tokens, redeem2.solverTip, redeem2.deadline, redeem2.maxPriceAge, true
        );


        _approveToken(users.eve, token, address(provisioner), redeem1.tokens + redeem2.tokens);

        Request[] memory requests = new Request[](2);
        requests[0] = redeem1;
        requests[1] = redeem2;

        vm.expectEmit(true, false, false, true, address(provisioner));
        emit IProvisioner.RedeemSolved(provisioner.getRequestHash(token, redeem1));
        vm.expectEmit(true, false, false, true, address(provisioner));
        emit IProvisioner.RedeemSolved(provisioner.getRequestHash(token, redeem2));

        vm.prank(users.eve);
        provisioner.solveRequestsDirect(token, requests);
        vm.snapshotGasLastCall("solveRequestsDirect - success - all solvable");

        assertFalse(provisioner.asyncRedeemHashes(provisioner.getRequestHash(token, redeem1)));
        assertFalse(provisioner.asyncRedeemHashes(provisioner.getRequestHash(token, redeem2)));

        uint256 eve_balanece_after_She_solves_in_underlying = token.balanceOf(users.eve);
        uint256 eve_balanece_after_She_solves_in_vault = IERC20(address(multiDepositorVault)).balanceOf(users.eve);
        

        Request memory redeem3 = _makeRequest({
            user: users.eve,
            requestType: RequestType.REDEEM_FIXED_PRICE,
            tokens: (eve_balanece_after_She_solves_in_vault - eve_balanece_before_She_solves_in_vault) * 9 / 10, // 10% as it is worth for the other solvers to call solveRequestsDirect
            units:  eve_balanece_after_She_solves_in_vault - eve_balanece_before_She_solves_in_vault,
            solverTip: 0,
            deadline: 1 days,
            maxPriceAge: UNIT_PRICE_AGE
        });


        Request[] memory requests2 = new Request[](1);
        requests2[0] = redeem3;

        ERC20Mock(address(token)).mint(users.dan, redeem3.units);
        _approveToken(users.dan, token, address(provisioner), redeem3.tokens);
        _approveToken(users.eve, IERC20(address(multiDepositorVault)), address(provisioner), redeem3.units);


        vm.prank(redeem3.user);
        provisioner.requestRedeem(
            token, redeem3.units, redeem3.tokens, redeem3.solverTip, redeem3.deadline, redeem3.maxPriceAge, true
        );


        vm.prank(users.dan);
        provisioner.solveRequestsDirect(token, requests2);
        vm.snapshotGasLastCall("solveRequestsDirect - success - all solvable");

        uint256 eve_balanece_after_She_redeem_in_underlying = token.balanceOf(users.eve);
        uint256 eve_balanece_after_She_redeem_in_vault = IERC20(address(multiDepositorVault)).balanceOf(users.eve);

        console.log("eve_balanece_before_She_solves_in_underlying in underlying:", eve_balanece_before_She_solves_in_underlying);
        console.log("eve_balanece_before_She_solves_in_vault in vault:", eve_balanece_before_She_solves_in_vault);
        console.log("eve_balanece_after_She_solves_in_underlying in underlying:", eve_balanece_after_She_solves_in_underlying);
        console.log("eve_balanece_after_She_solves_in_vault in vault:", eve_balanece_after_She_solves_in_vault);
        console.log("eve_balanece_after_She_redeem_in_underlying in underlying:", eve_balanece_after_She_redeem_in_underlying);
        console.log("eve_balanece_after_She_redeem_in_vault in vault:", eve_balanece_after_She_redeem_in_vault);
    }
```

Simplyfied example:

1. **User** holds 1,000 vault units. Calls:

   ```solidity
   provisioner.requestRedeem(
     WETH,           // underlying
     1000,           // unitsIn
     0.95 ETH worth, // minTokensOut
     0,              // solverTip = 0
     deadline,
     maxPriceAge,
     true            // fixed price
   );
   ```
2. **Solver** calls `solveRequestsDirect`:

   * Transfers 1,000 units → solver
   * Transfers WETH → user

3. **Solver** now has 1,000 vault units and has to pay a tip to another solver.

   
## Recommendation

Provide a **one‑transaction direct‐redeem‐and‐exit** path so that solvers receive underlying immediately.

**Benefits**:

* Solver never holds illiquid vault tokens—they’re burned in the same transaction by the exit functionality inside `MultiDepositorVault`.
* They instantly receive underlying, aligning incentives.

This way the solver does not have to pay extra gas fees and tips to other solvers for him to get his underlying token.


## [M-2] Small-tip requests makes funds to be locked until deadline, 0 tip makest it possible to refund imidietly

## Summary

Users who submit an async deposit or redeem request with a long deadline (up to 356 days) and offer a small solver tip can have their funds locked in the Provisioner contract until the deadline elapses. During this time, non-authorized users cannot cancel or refund their own requests, creating a denial-of-service on their own assets.  Also if the `depositCap` is nearly going to be hith the request can not be solved if the deposit is too big and again the funds are locked with no way of getting them before the deadline.

On the other hand having `solverFee = 0` makes it possibleto refund in a `directRequest`.

## Finding Description

The `refundRequest` function only allows refunds when `request.deadline < block.timestamp` or when the caller is authorized:

```solidity
require(
  request.deadline < block.timestamp || isAuthorized(msg.sender, msg.sig),
  Aera__DeadlineInFutureAndUnauthorized()
);
```
However this actualy gets broken when the user wants to make a `directRedeem` or `directDeposit` with `solverFee = 0`. A user that has made a request with 0 solver fee can call `solveRequestsDirect` on his own request which will basically refund his request before the `request.deadline` has passed. On the other hand if a user creates a request with a low `solverFee` and a big `deadline` his funds are going to be basically locked as no solver would want to solve his request and the function `solveRequestsVault` is permisioned and the user can not call it on his own as it is possible with `solveRequestsDirect`.

## Impact Explanation

**If a user creates a request with `solverFee = 0`**: Users can refund their request through `solveRequestsDirect` which breaks the asumption that `refundRequest` function only allows refunds when `request.deadline < block.timestamp`.

**If a user creates a request with a low `solverFee` and a big `deadline` or the depositCap is nearly going to be hit**: Users’ funds are frozen for up to a year without recourse. They cannot reallocate capital, redeem, or correct mistaken parameters. This creates financial risk and a poor user experience.

## Likelihood Explanation

Every time a request is made with `solverFee = 0` a user can imidiately refund it.
If `solverFee` is low most likely his request is not going to be executed for the whole period.

## Proof of Concept

1. Alice calls `requestDeposit(token, 100, minUnits, 0, deadline = now + 300 days, …)`.
2. She can imediatly refund her request through `solveRequestsDirect`.

You can add this test to `Provisioner.t.sol` to see that `refundRequest` fails but, by calling `solveRequestsDirect` the user ends you with the same balance as he was before the request.
Add `import "forge-std/console.sol";` and run the test with forge `test --mt test_solveRequestsDirect_success_all_solvable -vvvv`.
```solidity
    function test_solveRequestsDirect_success_all_solvable() public {
        vm.prank(users.owner);
        provisioner.setTokenDetails(
            token,
            TokenDetails({
                syncDepositEnabled: false,
                asyncDepositEnabled: true,
                asyncRedeemEnabled: true,
                depositMultiplier: uint16(ONE_IN_BPS),
                redeemMultiplier: uint16(ONE_IN_BPS)
            })
        );

        Request memory redeem1 = _makeRequest({
            user: users.charlie,
            requestType: RequestType.REDEEM_FIXED_PRICE,
            tokens: 3000 ether,
            units: 3300 ether,
            solverTip: 0,
            deadline: 1 days,
            maxPriceAge: UNIT_PRICE_AGE
        });

        multiDepositorVault.mint(redeem1.user, redeem1.units);
        _approveToken(redeem1.user, IERC20(address(multiDepositorVault)), address(provisioner), redeem1.units);

        ERC20Mock(address(token)).mint(users.charlie, redeem1.tokens);

        uint256 charlie_balanece_before_She_solves_in_underlying = token.balanceOf(users.charlie);
        uint256 charlie_balanece_before_She_solves_in_vault = IERC20(address(multiDepositorVault)).balanceOf(users.charlie);

        // Make redeem requests
        vm.prank(redeem1.user);
        provisioner.requestRedeem(
            token, redeem1.units, redeem1.tokens, redeem1.solverTip, redeem1.deadline, redeem1.maxPriceAge, true
        );

        _approveToken(users.charlie, token, address(provisioner), redeem1.tokens);

        Request[] memory requests = new Request[](1);
        requests[0] = redeem1;

        vm.prank(users.charlie);
        vm.expectRevert();
        provisioner.refundRequest(token, redeem1);

        uint256 charlie_balanece_after_request_before_solve_in_underlying = token.balanceOf(users.charlie);
        uint256 charlie_balanece_after_request_before_solve_in_vault = IERC20(address(multiDepositorVault)).balanceOf(users.charlie);

        vm.prank(users.charlie);
        provisioner.solveRequestsDirect(token, requests);
        vm.snapshotGasLastCall("solveRequestsDirect - success - all solvable");

        uint256 charlie_balanece_after_She_solves_in_underlying = token.balanceOf(users.charlie);
        uint256 charlie_balanece_after_She_solves_in_vault = IERC20(address(multiDepositorVault)).balanceOf(users.charlie);
        

        console.log("charlie_balanece_before_She_solves_in_underlying in underlying:", charlie_balanece_before_She_solves_in_underlying);
        console.log("charlie_balanece_before_She_solves_in_vault in vault:", charlie_balanece_before_She_solves_in_vault);
        console.log("charlie_balanece_after_request_before_solve_in_underlying in underlying:", charlie_balanece_after_request_before_solve_in_underlying);
        console.log("charlie_balanece_after_request_before_solve_in_vault in vault:", charlie_balanece_after_request_before_solve_in_vault);
        console.log("charlie_balanece_after_She_solves_in_underlying in underlying:", charlie_balanece_after_She_solves_in_underlying);
        console.log("charlie_balanece_after_She_solves_in_vault in vault:", charlie_balanece_after_She_solves_in_vault);
    }
```

1. Alice calls `requestDeposit(token, 100, minUnits, 100, deadline = now + 300 days, …)`.
2. No solver ever calls `solveRequestsVault`, because tip = 100 and thay would win nothing.
3. Only after 300 days can Alice call `refundRequest` herself and recover funds.

And for this situation it is clear that there is no method of inlocking the funds as `refundRequest`, `solveRequestsVault` and `solveRequestsDirect` would revert.

## Recommendation

Allow users to cancel their own requests immediately so thay can not lock themselves for a long time or if that is the intention make `solveRequestsDirect` permisioned.


## Disclamer:
The reason i chose the severity to be informational is because i did not find clear documentation abouth this, it might be a design but i have no way of finding out, please give me a feedback on if that is intended or it is a real issue.

As from what i see here https://docs.aera.finance/entry-exit-with-provisioner#caveats:~:text=If%20the%20deadline%20passes%20and%20the%20order%20wasn%27t%20filled%2C%20anyone%20can%20call%20refundRequest%20to%20refund%20the%20order :
> If the deadline passes and the order wasn't filled, anyone can call refundRequest to refund the order.
NOTE: Request redeem works analogously.

Before the deadline it should not be possible to refund.

