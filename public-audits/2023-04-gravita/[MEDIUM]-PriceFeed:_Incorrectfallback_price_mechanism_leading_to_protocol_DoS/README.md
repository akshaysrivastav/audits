# Original link
https://github.com/Gravita-Protocol/Gravita-SmartContracts/issues/238



# Vulnerability details

The PriceFeed contract intends to use a fallback price mechanism. When the latest ChainLink price is stale, the `_processFeedResponses` function tries to fallback to the stored `priceRecord.scaledPrice` value. But the implementation of this function is broken.

## Affected smart contract

- PriceFeed
```solidity
 bool isValidResponse = _isFeedWorking(_currResponse, _prevResponse) && 
 	!_isPriceStale(_currResponse.timestamp) && 
 	!_isPriceChangeAboveMaxDeviation(oracle.maxDeviationBetweenRounds, _currResponse, _prevResponse); 
  
 if (isValidResponse) { 
 	uint256 scaledPrice = _scalePriceByDigits(uint256(_currResponse.answer), _currResponse.decimals); 
 	if (oracle.isEthIndexed) { 
 		// Oracle returns ETH price, need to convert to USD 
 		scaledPrice = _calcEthPrice(scaledPrice); 
 	} 
 	if (!oracle.isFeedWorking) { 
 		_updateFeedStatus(_token, oracle, true); 
 	} 
 	_storePrice(_token, scaledPrice, _currResponse.timestamp); 
 	return scaledPrice; 
 } else { 
 	if (oracle.isFeedWorking) { 
 		_updateFeedStatus(_token, oracle, false); 
 	} 
 	PriceRecord memory priceRecord = priceRecords[_token]; 
 	if (_isPriceStale(priceRecord.timestamp)) { 
 		revert PriceFeed__FeedFrozenError(_token); 
 	} 
 	return priceRecord.scaledPrice; 
 } 
```

## Description
In the `_processFeedResponses` function, the `_currResponse` is validated against staleness.

```solidity
    bool isValidResponse = _isFeedWorking(_currResponse, _prevResponse) && 
        !_isPriceStale(_currResponse.timestamp) && 
        !_isPriceChangeAboveMaxDeviation(oracle.maxDeviationBetweenRounds, _currResponse, _prevResponse); 
```

In case the `_currResponse` is stale the `isValidResponse` boolean turns out to be `false`. Hence the `else` block is executed.

```solidity
    if (isValidResponse) {
        ...
    } else {
        ...
        PriceRecord memory priceRecord = priceRecords[_token];
        if (_isPriceStale(priceRecord.timestamp)) {
            revert PriceFeed__FeedFrozenError(_token);
        }
        return priceRecord.scaledPrice;
    }
```

It should be noted that the `else` block also checks the staleness of previously recorded `priceRecord` value, if `priceRecord` value is stale the txn is reverted.

This is where the issue lies. Chainlink oracles work with incremental `roundIds`. Round id `n + 1` always comes after round id `n` with regards to timestamps.

Hence when the `_currResponse` is stale, it is almost certain that the previously recorded `priceRecord` will also be stale. And as the staleness check is also performed for `priceRecord` the `fetchPrice` call will get reverted. Essentially the code in the `else` will never get executed successfully when `_currResponse` is stale.

This issue causes DoS for the `fetchPrice` function. This function is used in other protocol functions like:

- BorrowerOperations.openVessel()
- BorrowerOperations.adjustVessel()
- BorrowerOperations.closeVessel()
- StabilityPool.withdrawFromSP()
- VesselManagerOperations.liquidateVessels()
- VesselManagerOperations.batchLiquidateVessels()
- VesselManagerOperations.redeemCollateral()

All these functions will suffer a DoS due to this bug.

Special attention should be given to the DoS of liquidation operations. As liquidations are crucial time sensitive operations for Gravita protocol, DoS of liquidations will lead to protocol solvency issues in case liquidations do not happen in timely manner.

Also to further alleviate the issue, the oracle configuration can only be updated by the timelock contract which has a minimum delay of two days. So DoS will happen until the oracle config is updated in the PriceFeed contract (minimum 2 days).

## Attack scenario
Consider this scenario:

- Chainlink report correct and fresh price `X` for a token, this valid price gets stored in `priceRecord`.
- Due to any possible reason, the latest price on Chainlink becomes stale (older than 4 hours).
- The `PriceFeed.fetchPrice` is called, due to a stale price from Chainlink the `isValidResponse` becomes `false`.
- The PriceFeed tries to fallback upon the previously recorded `priceRecord` value. The `else` block is executed of `_processFeedResponses` function.
- The staleness check is performed again for `priceRecord` value which fails (reasons already explained)
- The `fetchPrice` call is reverted along with the parent call (like liquidation).

## Recommendation
Do not perform the same `RESPONSE_TIMEOUT` duration staleness check for both `_currResponse` and `priceRecord`.

There could be multiple mitigation paths here like:

performing the `priceRecord` staleness check with much longer duration than `RESPONSE_TIMEOUT`
omitting the staleness check for `priceRecord` (this could be risky)
creating some other fallback mechanism like using some other fallback oracle than chainlink.
