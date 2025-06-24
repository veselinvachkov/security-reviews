## Calls to Oracles don't check for stale prices

## Summary
**Severity Medium**

The function `getPriceUSDInWad` in `napier-v2\src\lens\Lens.sol` does not check whether the price data retrieved from the oracle is stale, potentially leading to outdated or incorrect valuations.

## Finding Description

The current implementation of the function retrieves price data from either a Chainlink FeedRegistry or a manually registered oracle. However, the function does not verify if the returned data is fresh by checking the `updatedAt` timestamp. Without this validation, the function might return outdated prices, leading to financial risks such as incorrect asset valuations, mispricing, and potential manipulation.

A stale price can occur if the oracle experiences an issue or stops updating, leading to decisions based on outdated data. An attacker could exploit this by manipulating market conditions based on old prices.

## Impact Explanation

The impact of this issue is **High**, as outdated price data could result in severe financial losses in automated financial systems relying on accurate pricing for lending, trading, and collateral calculations. Mispricing could also lead to arbitrage opportunities that drain protocol funds.

## Likelihood Explanation

The likelihood of this occurring is **Low**, as it depends on external factors such as oracle outages or network congestion. However, since smart contracts often rely on external data, it is a common scenario that must be accounted for.

## Exploit

Consider the following scenario:

1. The oracle stops updating price data due to an outage.
2. The contract continues to fetch the last available price without checking if it is fresh.
3. Users interact with the contract under the assumption that the price is up-to-date, leading to inaccurate valuations and potential financial losses.

Example snippet demonstrating the missing check:

```solidity
// If FeedRegistry is available, use it. Otherwise, try to use registered price oracle.
        if (address(s_feedRegistry) != address(0)) {
            decimals = s_feedRegistry.decimals(currency, Denominations.USD);
            (
                /* uint80 roundId */
                ,
                answer,
                /* uint256 startedAt */
                ,
                /* uint256 updatedAt */
                ,
                /* uint80 answeredInRound */
            ) = s_feedRegistry.latestRoundData({base: currency, quote: Denominations.USD});
        } else {
            AggregatorV3Interface oracle = s_priceOracles[currency];
            if (address(oracle) == address(0)) return 0;

            decimals = oracle.decimals();
            (
                /* uint80 roundId */
                ,
                answer,
                /* uint256 startedAt */
                ,
                /* uint256 updatedAt */
                ,
                /* uint80 answeredInRound */
            ) = oracle.latestRoundData();
        }
```
Here is also a similar bug found in a diferent project: https://solodit.cyfrin.io/issues/m-1-calls-to-oracles-dont-check-for-stale-prices-sherlock-none-ussd-autonomous-secure-dollar-git

## Proof of Concept
- The function passes because the outdated price is not detected.
- You can add the function in napier-v2/test/lens/Lens.t.sol
```solidity
function test_StaleOraclePrice() public {
        // Mock an outdated timestamp (e.g., 1 year old)
        uint256 staleTimestamp = block.timestamp - 365 days;
        int256 stalePrice = 3000 * 10**8; // Simulate $3000 price with 8 decimals
        uint256 decimals = 8;

        // Mock feed registry response with stale data
        vm.mockCall(
            mockFeedRegistry,
            abi.encodeWithSelector(FeedRegistry.decimals.selector),
            abi.encode(decimals)
        );
        vm.mockCall(
            mockFeedRegistry, 
            abi.encodeWithSelector(FeedRegistry.latestRoundData.selector),
            abi.encode(0, stalePrice, 0, staleTimestamp, 0)
        );

        // Fetch the price, which should not detect stale data
        uint256 price = lens.getPriceUSDInWad(address(base));
        
        // Check if price is still returned despite being stale
        assertEq(price, uint256(stalePrice) * 10 ** (18 - decimals), "Stale price should not be used");
    }
```

## Recommendation
To mitigate this risk, it is recommended to implement a check to verify that the price data is recent. This can be done by comparing `updatedAt` with `block.timestamp` to ensure it falls within an acceptable time range(The curent block of the transaction).

Suggested fix:
```solidity
// Fetch the latest round data
(
    /* uint80 roundId */,
    answer,
    /* uint256 startedAt */,
    uint256 updatedAt,
    /* uint80 answeredInRound */
) = oracle.latestRoundData();

// Ensure price is recent and from a valid round
require(updatedAt > block.timestamp - 60 * 60 /* 1 hour */, "Price data is stale");
```
It would also be even better if you implemented a check to see if `roundId == answeredInRound`, but this is not necessary.
This change ensures that the function only returns prices that are recent, helping to prevent outdated data from being used in financial operations.