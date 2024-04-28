Magic Orchid Perch

medium

# `AerodromePoolAM` LP Position should not default to zero if only one underlying `assetValue` is zero


## Summary

`AerodromePoolAM` is the asset module for managing LP tokens for Aerodrome AMM Pools. Each LP token represents 2 different underlying tokens. The value of the LP position is calculated by the sum of the products of the amounts of each token and their respective `assetValue` (determined by an oracle). However, if the oracle reports a value of zero for only **one** underlying token, the total value of the LP position defaults to zero, and not counting the value of the other underlying token. This results in an underestimation of the collateral in an account.

## Vulnerability Detail

In `AerodromePoolAM`, there are two underlying assets for each LP token. When calculating the value for the LP position, it does the following:

1. Get oracle price `assetValue` for underlying token0 and token1.
2. Calculate the "trusted reserves" of the AMM pool for token0 and token1 (the actual reserve is not used to avoid sandwich attacks)
3. Calculate the amount of token0 and token1
4. The final `valudInUsd` for this position is: `amount0 * assetValue0 + amount1 * assetValue1` (simplified).

The issue here is that when the oracle returns `assetValue == 0` for only **one** underlying asset, the function `_getUnderlyingAssetsAmounts()` returns zero for both assets, leading to a value of zero for the entire LP position. This is incorrect, because for the other underlying asset, it still may have a non-zero `assetValue`, and the value of LP position should be equal to the worth of the that asset, which can be calculated using the actual reserve of the AMM pool.

AerodromePoolAM.sol:
```solidity
    function _getUnderlyingAssetsAmounts(
        address creditor,
        bytes32 assetKey,
        uint256 amount,
        bytes32[] memory underlyingAssetKeys
    )
        internal
        view
        override
        returns (uint256[] memory underlyingAssetsAmounts, AssetValueAndRiskFactors[] memory rateUnderlyingAssetsToUsd)
    {
        underlyingAssetsAmounts = new uint256[](2);
        rateUnderlyingAssetsToUsd = _getRateUnderlyingAssetsToUsd(creditor, underlyingAssetKeys);

>       // If one of the assets has a rate of 0, the whole LP positions will have a value of zero.
>       if (rateUnderlyingAssetsToUsd[0].assetValue == 0 || rateUnderlyingAssetsToUsd[1].assetValue == 0) {
>           return (underlyingAssetsAmounts, rateUnderlyingAssetsToUsd);
>       }

        (address pool,) = _getAssetFromKey(assetKey);

        // Calculate the trusted reserves of the pool.
        (uint256 trustedReserve0, uint256 trustedReserve1) = assetToInformation[pool].stable
            ? _getTrustedReservesStable(pool, rateUnderlyingAssetsToUsd)
            : _getTrustedReservesVolatile(pool, rateUnderlyingAssetsToUsd);

        // Cache totalSupply
        uint256 totalSupply = IAeroPool(pool).totalSupply();

        underlyingAssetsAmounts[0] = trustedReserve0.mulDivDown(amount, totalSupply);
        underlyingAssetsAmounts[1] = trustedReserve1.mulDivDown(amount, totalSupply);

        return (underlyingAssetsAmounts, rateUnderlyingAssetsToUsd);
    }
```

AbstractDerivedAM.sol:
```solidity
    function getValue(address creditor, address asset, uint256 assetId, uint256 assetAmount)
        public
        view
        virtual
        override
        returns (uint256 valueInUsd, uint256 collateralFactor, uint256 liquidationFactor)
    {
        bytes32 assetKey = _getKeyFromAsset(asset, assetId);
        bytes32[] memory underlyingAssetKeys = _getUnderlyingAssets(assetKey);

>       (uint256[] memory underlyingAssetsAmounts, AssetValueAndRiskFactors[] memory rateUnderlyingAssetsToUsd) =
>           _getUnderlyingAssetsAmounts(creditor, assetKey, assetAmount, underlyingAssetKeys);

        // Check if rateToUsd for the underlying assets was already calculated in _getUnderlyingAssetsAmounts().
        if (rateUnderlyingAssetsToUsd.length == 0) {
            // If not, get the USD value of the underlying assets recursively.
            rateUnderlyingAssetsToUsd = _getRateUnderlyingAssetsToUsd(creditor, underlyingAssetKeys);
        }

>       (valueInUsd, collateralFactor, liquidationFactor) =
>           _calculateValueAndRiskFactors(creditor, underlyingAssetsAmounts, rateUnderlyingAssetsToUsd);
    }
```

AerodromePoolAM.sol:
```solidity
    function _calculateValueAndRiskFactors(
        address creditor,
        uint256[] memory underlyingAssetsAmounts,
        AssetValueAndRiskFactors[] memory rateUnderlyingAssetsToUsd
    ) internal view override returns (uint256 valueInUsd, uint256 collateralFactor, uint256 liquidationFactor) {
        // "rateUnderlyingAssetsToUsd" is the USD value with 18 decimals precision for 10**18 tokens of Underlying Asset.
        // To get the USD value (also with 18 decimals) of the actual amount of underlying assets, we have to multiply
        // the actual amount with the rate for 10**18 tokens, and divide by 10**18.
>       valueInUsd = underlyingAssetsAmounts[0].mulDivDown(rateUnderlyingAssetsToUsd[0].assetValue, 1e18)
>           + underlyingAssetsAmounts[1].mulDivDown(rateUnderlyingAssetsToUsd[1].assetValue, 1e18);

        // Lower risk factors with the protocol wide risk factor.
        uint256 riskFactor = riskParams[creditor].riskFactor;

        // Keep the lowest risk factor of all underlying assets.
        collateralFactor = rateUnderlyingAssetsToUsd[0].collateralFactor < rateUnderlyingAssetsToUsd[1].collateralFactor
            ? riskFactor.mulDivDown(rateUnderlyingAssetsToUsd[0].collateralFactor, AssetValuationLib.ONE_4)
            : riskFactor.mulDivDown(rateUnderlyingAssetsToUsd[1].collateralFactor, AssetValuationLib.ONE_4);
        liquidationFactor = rateUnderlyingAssetsToUsd[0].liquidationFactor
            < rateUnderlyingAssetsToUsd[1].liquidationFactor
            ? riskFactor.mulDivDown(rateUnderlyingAssetsToUsd[0].liquidationFactor, AssetValuationLib.ONE_4)
            : riskFactor.mulDivDown(rateUnderlyingAssetsToUsd[1].liquidationFactor, AssetValuationLib.ONE_4);
    }
```

## Impact

This will lead to a decrease in account collateral, and may cause accounts to be liquidated when they shouldn't have been.

## Code Snippet

- https://github.com/sherlock-audit/2024-04-arcadia-pricing-module/blob/main/accounts-v2/src/asset-modules/Aerodrome-Finance/AerodromePoolAM.sol#L186-L188
- https://github.com/sherlock-audit/2024-04-arcadia-pricing-module/blob/main/accounts-v2/src/asset-modules/abstracts/AbstractDerivedAM.sol#L199-L200

## Tool used

Manual review

## Recommendation

Two mitigations:

1. In `_getUnderlyingAssetsAmounts()`, return 0 if only both `assetValue` are 0. If only one `assetValue` is 0, simply use the actual reserve of the AMM pool to calculate the amount of tokens.
2. Simply revert upon a `assetValue` equal to 0, as most likely it is an oracle issue.
