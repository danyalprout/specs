# Predeploys

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [GasPriceOracle](#gaspriceoracle)
  - [L1 Gas Usage Estimation](#l1-gas-usage-estimation)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## GasPriceOracle

Following the Fjord upgrade, three additional values used for L1 fee computation are:

- costIntercept
- costFastlzCoef
- minTransactionSize

These values are hard-coded constants in the `GasPriceOracle` contract. The
calculation follows the same formula outlined in the
[Fjord L1-Cost fee changes (FastLZ estimator)](./exec-engine.md#fjord-l1-cost-fee-changes-fastlz-estimator)
section.

A new method is introduced: `getL1FeeUpperBound(uint256)`. This method returns an upper bound for the L1 fee
for a given transaction size. It is provided for callers who wish to estimate L1 transaction costs in the
write path, and is much more gas efficient than `getL1Fee`.

The upper limit overhead is assumed to be `original/255+16`, borrowed from LZ4. According to historical data, this
approach can encompass more than 99.99% of transactions.

This is implemented as follows:

```solidity
function getL1FeeUpperBound(uint256 unsignedTxSize) external view returns (uint256) {
    uint256 l1FeeScaled = baseFeeScalar() * l1BaseFee() * 16 + blobBaseFeeScalar() * blobBaseFee();
    
    // txSize / 255 + 16 is the pratical fastlz upper-bound covers 99.99% txs.
    // Add 68 to account for unsigned tx
    int256 flzUpperBound = int256(unsignedTxSize) + int256(unsignedTxSize) / 255 + 16 + 68;
    
    int256 estimatedSize = costIntercept + costFastlzCoef * flzUpperBound;
    if (estimatedSize < minTransactionSize) {
        estimatedSize = minTransactionSize;
    }
    
    return uint256(estimatedSize) * feeScaled / (10 ** (DECIMALS * 2));
}
```

### L1 Gas Usage Estimation

The `getL1GasUsed` method is updated to take into account the improved [compression estimation](./exec-engine.md#fees)
accuracy as part of the Fjord upgrade.

```solidity
function getL1GasUsed(bytes memory _data) public view returns (uint256) {
    if (isFjord) {
        // add 68 to the size to account for unsigned tx:
        uint256 estimatedSize = LibZip.flzCompress(_data).length + 68;
        if (estimatedSize < MIN_TRANSCTION_SIZE) {
            estimatedSize = MIN_TRANSCTION_SIZE;
        }

        // Due to the minimum fastlz size check, it's not possible for cost to be a negative number
        return COST_INTERCEPT + int256(COST_FASTLZ_COEF * estimatedSize);
    }
    // ...
}
```

The `getL1GasUsed` method will be deprecated. This is due to it not accurately estimating the
L1 gas used, for a transaction.

Users can continue to use the `getL1FeeUpperBound` or `getL1Fee` method to estimate the L1 fee for a given transaction.