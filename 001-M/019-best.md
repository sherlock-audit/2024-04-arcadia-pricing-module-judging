Interesting Iron Moth

medium

# `addAsset` will store wrong decimals if the underlying token less than 18.

## Summary
`addAsset` will store wrong decimals
## Vulnerability Detail
> Please raise issues regarding the underlying tokens that can currently be used: WETH, DAI, COMP, USDBC, USDC, CBETH, RETH, STG, wstETH

the contract use above tokens as underlying, but there is one token had decimals of 6! (USDBC) .

the function `addAsset` store decimals as it's 18. but USDbC had 6  decimals . this will lead to wrong calculated in the contract . 
## Impact
When the decimals of one or both tokens in the pair is not 18, the price will be way off.
## Code Snippet
https://github.com/sherlock-audit/2024-04-arcadia-pricing-module/blob/54832626b806dcb72e250935fc193acb95e0cf8a/accounts-v2/src/asset-modules/Aerodrome-Finance/AerodromePoolAM.sol#L76C5-L105C6
```solidity
function addAsset(address pool) external {
        if (AERO_FACTORY.isPool(pool) != true) revert InvalidPool();

        (address token0, address token1) = IAeroPool(pool).tokens();
        if (!IRegistry(REGISTRY).isAllowed(token0, 0)) revert AssetNotAllowed();
        if (!IRegistry(REGISTRY).isAllowed(token1, 0)) revert AssetNotAllowed();

        if (IAeroPool(pool).stable()) {
            // Only owner can add Stable pools, since tokens with very high supply (>15511800964 * 10 ** decimals)
            // might cause an overflow in _getTrustedReservesStable().
            if (msg.sender != owner) revert OnlyOwner();

            assetToInformation[pool] = AssetInformation({
                stable: true,
                unitCorrection0: uint64(10 ** (18 - ERC20(token0).decimals())), //@audit
                unitCorrection1: uint64(10 ** (18 - ERC20(token1).decimals()))
            });
        }

        inAssetModule[pool] = true;

        bytes32[] memory underlyingAssetsKey = new bytes32[](2);
        underlyingAssetsKey[0] = _getKeyFromAsset(token0, 0);
        underlyingAssetsKey[1] = _getKeyFromAsset(token1, 0);

        assetToUnderlyingAssets[_getKeyFromAsset(pool, 0)] = underlyingAssetsKey;

        // Will revert in Registry if Aerodrome Finance pool was already added.
        IRegistry(REGISTRY).addAsset(uint96(ASSET_TYPE), pool);
    }
```
## Tool used

Manual Review

## Recommendation
remove or add support to USDbC.