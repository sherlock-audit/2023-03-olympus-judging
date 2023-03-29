Bahurum

high

# Incorrect OHM price calculation

## Summary
In `getOhmTknPrice` and `getTknOhmPrice` functions in `BLVaultManagerLido.sol` the `stethPerWsteth` is used as it is the value of wstETH in stETH while it is actually the value of stETH in wstETH. 

## Vulnerability Detail
`IWsteth(pairToken).stEthPerToken()` reuturns the amount of `wstEth` corresponding to 1 `stETH`. For example right now it is 0.896. Let's say `ethPerOhm`=0.00647 and `stethPerEth`=0.99

In `getOhmTknPrice` the formula returns

0.896*0.99/0.00647 = 137.1

meaning 137.1 OHM per wstETH. If we compute the OHM/wstETM ratio from the market price: OHM = 10.30 USD, wstETH = 1962.5 USD --> 190.5. 

In a similar way, the value returned by `getTknOhmPrice` is larger than expected. 


## Impact
Oracle prices are incorrect: this will cause the amount of OHM deposited into the balancer pool to be unbalanced (asymmetric liquidity deposit) so depositors will get back only a fraction of the wstETH deposited when withdrawing.

## Code Snippet
https://github.com/sherlock-audit/2023-03-olympus/blob/main/sherlock-olympus/src/policies/BoostedLiquidity/BLVaultManagerLido.sol#L454

https://github.com/sherlock-audit/2023-03-olympus/blob/main/sherlock-olympus/src/policies/BoostedLiquidity/BLVaultManagerLido.sol#L472
## Tool used

Manual Review

## Recommendation
In `getOhmTknPrice()`:

```solidity
return (stethPerEth * 1e27) / (ethPerOhm * stethPerWsteth);
```

In `getTknOhmPrice()`:

```solidity
return (ethPerOhm * stethPerWsteth) / stethPerEth;
```