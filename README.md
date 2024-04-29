# Issue M-1: Donating (and syncing) tokens to an Aerodrome allows to bypass exposure limits 

Source: https://github.com/sherlock-audit/2024-04-arcadia-pricing-module-judging/issues/17 

## Found by 
zzykxx
## Summary
An attacker can bypass the exposure limits by donating tokens to an Aerodrome pool in which he owns ~100% of the liquidity, this will inflate reserves which will increase the value of the collateral and the attacker won't lose capital because he owns ~100% of the liquidity.

## Vulnerability Detail
Attack scenario:
1. Create a new Aerodrome pool, let's suppose an STG/WETH pool
2. Provide liquidity to the pool and mint LPs, let's suppose we mint ~1000$ worth of liquidity
3. Deposit the LP in an Arcadia account, the collateral is worth ~1000$
4. Send STG and WETH directly to the Aerodrome pool, let's suppose we send ~500_000$ worth of tokens
5. Call [sync()](https://github.com/aerodrome-finance/contracts/blob/main/contracts/Pool.sol#L397-L399) on the Aerodrome pool
6. The pool reserves are increased and the collateral is now worth ~501_000$

By doing this an attacker can bypass exposure limits. Note that Arcadia allows the use of any volatile Aerodrome pool LPs as collateral, as long as the underlying tokens are allowed.

## Impact
An attacker can bypass the exposure limits set by a creditor. 

## Code Snippet

## Tool used

Manual Review

## Recommendation



## Discussion

**sherlock-admin4**

1 comment(s) were left on this issue during the judging contest.

**shealtielanz** commented:
>  dup of #018



**Thomas-Smets**

Valid, think it is not a duplicate of 18 tho

**sherlock-admin3**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/arcadia-finance/accounts-v2/pull/201


# Issue M-2: Swapping large amounts of assets back and forth in an Aerodrome pool allows to bypass exposure limits 

Source: https://github.com/sherlock-audit/2024-04-arcadia-pricing-module-judging/issues/18 

## Found by 
PUSH0, zzykxx
## Summary
Swapping large amounts of assets back and forth in an Aerodrome pool allows to bypass exposure limits. 

## Vulnerability Detail
The `WrappedAerodromeAM` allows wrapping LPs of an Aerodrome pool to then deposit the wrapped position as collateral in an Arcadia account. The wrapper accounts for the [trading fees accumulated by the LP](https://github.com/sherlock-audit/2024-04-arcadia-pricing-module/blob/main/accounts-v2/src/asset-modules/Aerodrome-Finance/WrappedAerodromeAM.sol#L182-L187) when estimating the value of the collateral:
```solidity
(, uint256 positionId) = _getAssetFromKey(assetKey);
(uint256 fee0Position, uint256 fee1Position) = feesOf(positionId);

underlyingAssetsAmounts = new uint256[](3);
underlyingAssetsAmounts[0] = positionState[positionId].amountWrapped;
underlyingAssetsAmounts[1] = fee0Position;
underlyingAssetsAmounts[2] = fee1Position;
```

It's possible to take advantage of this to bypass exposure limits. An attacker can:
1. Create a new Aerodrome pool
2. Mint LPs, the attacker owns the whole liquidity supply
3. Deposit the LP as collateral via `WrappedAerodromeAM`
4. Swap large amounts in the Aerodrome pool back and forth to increase the fees accumulated by the LP. The value of the collateral increases while the collateral is already deposited. The attacker won't lose anything besides gas fees because he controls the whole (minues 1e3) liquidity supply.

An attacker can keep swapping amounts indefinitely. Because this is done while the collateral is already deposited in the account the attacker can avoid the restrictions of exposure limits.

## Impact
Exposure limits can be bypassed.

## Code Snippet

## Tool used

Manual Review

## Recommendation
A similar attack is possible on UniswapV3 positions, however in that case the protocol already implements safety guards against this possibility. Implement a similar safety guard for Aerodrome positions as well.




## Discussion

**sherlock-admin4**

1 comment(s) were left on this issue during the judging contest.

**shealtielanz** commented:
>  exposure limits will truly be bypassed



**sherlock-admin3**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/arcadia-finance/accounts-v2/pull/201


# Issue M-3: WrappedAerodromeAM.sol is not compatible with the Revert on Zero Value Tokens 

Source: https://github.com/sherlock-audit/2024-04-arcadia-pricing-module-judging/issues/23 

## Found by 
blackhole, merlin
## Summary
From the README file:
We also want to receive issues regarding minimal implementations of ERC20 Tokens, which besides a standard implementation have one of the following "weird behaviours": `Revert on Zero Value`.

## Vulnerability Detail
When the owner of a position calls `WrappedAerodromeAM.decreaseLiquidity`, they should receive the transferred liquidity back and fees.

Let's consider the following scenario:

1)The owner of the position calls `decreaseLiquidity`, and they are supposed to receive fees.
```solidity
        // Claim any pending fees from the Aerodrome Pool.
        (uint256 fee0Pool, uint256 fee1Pool) = _claimFees(pool);

        // Calculate the new fee balances.
        (poolState_, positionState_) = _getFeeBalances(poolState_, positionState_, fee0Pool, fee1Pool);
```
2)Since the fee for the `revert on zero transfer token` is zero, a DOS will occur, and the user will not be able to decrease liquidity.
```solidity
        // Pay out the fees to the position owner.
        ERC20(token0[pool]).safeTransfer(msg.sender, fee0Position);  //fee0Position = 0
        ERC20(token1[pool]).safeTransfer(msg.sender, fee1Position); // or fee1Position = 0
        emit FeesPaid(positionId, uint128(fee0Position), uint128(fee1Position));
```

## Impact
A denial-of-service (DOS) can occur on the `decreaseLiquidity` and `claimFees` functions in `WrappedAerodromeAM` smart contract of the revert on zero transfer token when the fee is zero value.

## Code Snippet
[src/asset-modules/Aerodrome-Finance/WrappedAerodromeAM.sol#L452-L453](https://github.com/sherlock-audit/2024-04-arcadia-pricing-module/blob/main/accounts-v2/src/asset-modules/Aerodrome-Finance/WrappedAerodromeAM.sol#L452-L453)
[src/asset-modules/Aerodrome-Finance/WrappedAerodromeAM.sol#L411-L412](https://github.com/sherlock-audit/2024-04-arcadia-pricing-module/blob/main/accounts-v2/src/asset-modules/Aerodrome-Finance/WrappedAerodromeAM.sol#L411-L412)

## Tool used

Manual Review

## Recommendation
Consider changing `decreaseLiquidity` and `claimFees` functions as follows:
```diff
+ if (fee0Position != 0)
            ERC20(token0[pool]).safeTransfer(msg.sender, fee0Position);
+ if (fee1Position != 0)
            ERC20(token1[pool]).safeTransfer(msg.sender, fee1Position);
```



## Discussion

**sherlock-admin4**

2 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  this should be valid considering the readMe stated to report issues with the said(weird) behavior; medium(1)

**shealtielanz** commented:
>  dup of #033



**sherlock-admin3**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/arcadia-finance/accounts-v2/pull/202


