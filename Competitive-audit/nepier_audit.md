## Severtity: Medium

## Summary

The function `getPriceUSDInWad` in `napier-v2\src\lens\Lens.sol` does not check whether the price data retrieved from the oracle is stale, potentially leading to outdated or incorrect valuations.

## Finding Description

The current implementation of the function retrieves price data from either a Chainlink FeedRegistry or a manually registered oracle. However, the function does not verify if the returned data is fresh by checking the `updatedAt` timestamp. Without this validation, the function might return outdated prices, leading to financial risks such as incorrect asset valuations, mispricing, and potential manipulation.

A stale price can occur if the oracle experiences an issue or stops updating, leading to decisions based on outdated data. An attacker could exploit this by manipulating market conditions based on old prices.

## Impact Explanation

The impact of this issue is **High**, as outdated price data could result in severe financial losses in automated financial systems relying on accurate pricing for lending, trading, and collateral calculations. Mispricing could also lead to arbitrage opportunities that drain protocol funds.

## Likelihood Explanation

The likelihood of this occurring is **Medium**, as it depends on external factors such as oracle outages or network congestion. However, since smart contracts often rely on external data, it is a common scenario that must be accounted for.

## Proof of Concept

Consider the following scenario:

1. The oracle stops updating price data due to an outage.
2. The contract continues to fetch the last available price without checking if it is fresh.
3. Users interact with the contract under the assumption that the price is up-to-date, leading to inaccurate valuations and potential financial losses.

Example snippet demonstrating the missing check:

```solidity
(
    /* uint80 roundId */,
    answer,
    /* uint256 startedAt */,
    /* uint256 updatedAt */,
    /* uint80 answeredInRound */
) = oracle.latestRoundData();
```
Here is also a similar bug found in a diferent project: https://solodit.cyfrin.io/issues/m-1-calls-to-oracles-dont-check-for-stale-prices-sherlock-none-ussd-autonomous-secure-dollar-git
## Recommendation
To mitigate this risk, it is recommended to implement a check to verify that the price data is recent. This can be done by comparing updatedAt with block.timestamp to ensure it falls within an acceptable time range.

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
This change ensures that the function only returns prices that are recent, helping to prevent outdated data from being used in financial operations.