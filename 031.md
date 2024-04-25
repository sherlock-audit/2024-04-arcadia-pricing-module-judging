Exotic Gauze Bird

medium

# `SlipstreamAM::_getFeeAmounts()` fetches the liquidity of the position, allowing the owner to increase `USD` balance as it pleases

## Summary

`SlipstreamAM::_getFeeAmounts()` fetches the liquidity of position, instead of using the cached liquidity. Thus, users that have pending fees may increase the liquidity of their positions to instantly boost the `USD` value of their position, as it is pro-rata to `liquidity`.

## Vulnerability Detail

`SlipstreamAM::_getFeeAmounts()` gets the fees of a position using the actual liquidity of the position [in](https://github.com/sherlock-audit/2024-04-arcadia-pricing-module/blob/main/accounts-v2/src/asset-modules/Slipstream/SlipstreamAM.sol#L328) the pool. This is not the case for other asset moduels such as `WrappedAerodromeAM`, which [uses](https://github.com/sherlock-audit/2024-04-arcadia-pricing-module/blob/main/accounts-v2/src/asset-modules/Aerodrome-Finance/WrappedAerodromeAM.sol#L519-L527) the cached `totalWrapped`. Due to the fact that fees are calculated [pro-rata](https://github.com/sherlock-audit/2024-04-arcadia-pricing-module/blob/main/accounts-v2/src/asset-modules/Slipstream/SlipstreamAM.sol#L344-L350) to `liquidity`, this means that anyone may increase the `USD` value of an already deposited `assetId`, which can serve several different purposes within the protocol and may be used to game it.

## Impact

Incosistent and silent increase of the `USD` value of an account [up to](https://github.com/sherlock-audit/2024-04-arcadia-pricing-module/blob/main/accounts-v2/src/asset-modules/Slipstream/SlipstreamAM.sol#L222-L223) `principal0` and `principal1` which may be used to game the protocol.

## Code Snippet

SlipstreamAM::_getFeeAmounts()` [fetching](https://github.com/sherlock-audit/2024-04-arcadia-pricing-module/blob/main/accounts-v2/src/asset-modules/Slipstream/SlipstreamAM.sol#L328) the actual liquidity of a position instead of the cached one.
```solidity
    ...
    function _getFeeAmounts(uint256 id) internal view returns (uint256 amount0, uint256 amount1) {
        (
            ,
            ,
            address token0,
            address token1,
            int24 tickSpacing,
            int24 tickLower,
            int24 tickUpper,
            uint256 liquidity, // gas: cheaper to use uint256 instead of uint128. //@audit here
            uint256 feeGrowthInside0LastX128,
            uint256 feeGrowthInside1LastX128,
            uint256 tokensOwed0, // gas: cheaper to use uint256 instead of uint128.
            uint256 tokensOwed1 // gas: cheaper to use uint256 instead of uint128.
        ) = NON_FUNGIBLE_POSITION_MANAGER.positions(id);
    ...
```

## Tool used

Manual Review

Vscode

## Recommendation

Use the cached liquidity to prevent abuse from `accounts`.
```solidity
function _getFeeAmounts(uint256 id) internal view returns (uint256 amount0, uint256 amount1) {
    (
        ,
        ,
        address token0,
        address token1,
        int24 tickSpacing,
        int24 tickLower,
        int24 tickUpper,
        , // unnecessary
        uint256 feeGrowthInside0LastX128,
        uint256 feeGrowthInside1LastX128,
        uint256 tokensOwed0, // gas: cheaper to use uint256 instead of uint128.
        uint256 tokensOwed1 // gas: cheaper to use uint256 instead of uint128.
    ) = NON_FUNGIBLE_POSITION_MANAGER.positions(id);

    uint256 liquidity = uint128(assetToLiquidity[assetId]); // get liquidity
    ...
}
```
