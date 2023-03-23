ABA

medium

# `OHM` to `wstETH` price is incorrectly calculated

## Summary

The price of `OHM`, relative to `wstETH`, which is used throughout the project, is calculated incorrectly because of wrong Chainlink stream pair used.

## Vulnerability Detail

OHM to token price is calculated in the function `getOhmTknPrice`
https://github.com/sherlock-audit/2023-03-olympus/blob/main/sherlock-olympus/src/policies/BoostedLiquidity/BLVaultManagerLido.sol#L440
```Solidity
    function getOhmTknPrice() public view override returns (uint256) {
```

The function takes the following token price pairs:
- **stETH/wstETH** -> `stethPerWsteth`
- **ETH/OHM** -> `ethPerOhm`
- **stETH/ETH** -> `stethPerEth`

and combines them with the intent to translated into **OHM/wstETH**. 

https://github.com/sherlock-audit/2023-03-olympus/blob/main/sherlock-olympus/src/policies/BoostedLiquidity/BLVaultManagerLido.sol#L453-L454
```Solidity
    // Calculate OHM per wstETH (9 decimals)
    return (stethPerWsteth * stethPerEth) / (ethPerOhm * 1e9);
```

Here is the issue, because the correct pairs are not used and results in a different price then intended. To better illustrate this will replace the above formula with the token denominations only:


$$ result = {stethPerWsteth * stethPerEth \over ethPerOhm} $$

- take the `ethPerOhm` to the right for better visibility

$$  = stethPerWsteth * stethPerEth * \frac{1}{ethPerOhm} $$

- replace each variable with it's denominations

$$  = \frac{stETH}{wstETH} * \frac{stETH}{ETH} * \frac{1}{\frac{ETH}{OHM}} $$

- reverse the `ETH/OHM` fraction

$$  = \frac{stETH}{wstETH} * \frac{stETH}{ETH} * \frac{OHM}{ETH} $$

- do some basic, allowed, positional changes and highlighting the fraction of interest in parentheses

$$  = (\frac{OHM}{wstETH}) * \frac{stETH}{ETH} * \frac{stETH}{ETH} $$

It can be seen in the final result that the desired value `OHM/wstETH` would only be achieved if in the original formula we had `stethPerEth` replaces with a `ethPerSteth (ETH/stETH)` equivalent:

```Solidity
return (stethPerWsteth * ethPerSteth) / (ethPerOhm * 1e9);
```

## Impact

`OHM/wstETH` price is calculated incorrect. By not reflecting the correct price the protocol can suffer losses due to arbitrage/unexpected price fluctuations.

Instead of the stETH to ETH price stream, the correct stream would of been ETH to stETH. Since both are ETH equivalents they tend to not vary much, as such, the price fluctuations would not be sever but would still pose an issue.

## Code Snippet

## Tool used

Manual Review

## Recommendation

The simples solution would be to use a `ETH/stETH` Chainlink price stream, unfortunately one does not exist 
https://docs.chain.link/data-feeds/price-feeds/addresses/

Instead, use other, intermediate, price streams to arrive at the desired price.