RaymondFam

high

# Increased `getTknOhmPrice()` not catered for

## Summary
The contract checks on the withdrawn ratios of OHM and wstETH against the current oracle price and takes any wstETH shifted imbalance as a fee to the treasury but it fails to cater for `getTknOhmPrice()` increase in pricing.

When `getTknOhmPrice()` increases, the system can be gamed while making a flash arbitrage gain.

## Vulnerability Detail
Here is a typical scenario, assuming the pool has been initiated with total LP equal to sqrt(100_000 * 1_000) = 10_000. (Note: OHM: $15, wstETH: $1500 with the pool pricing match up with [manager.getOhmTknPrice()](https://github.com/sherlock-audit/2023-03-olympus/blob/main/sherlock-olympus/src/policies/BoostedLiquidity/BLVaultLido.sol#L156) or [manager.getTknOhmPrice()](https://github.com/sherlock-audit/2023-03-olympus/blob/main/sherlock-olympus/src/policies/BoostedLiquidity/BLVaultLido.sol#L232), i.e. 100 OHM to 1 wstETH or 0.01 wstETH to 1 OHM. The pool token balances in each step below may be calculated via the [Constant Product Simulation](https://amm-calculator.vercel.app/) after each swap and stake.)

    OHM token balance: 100_000
    wstETH token balance: 1_000
    Total LP: 10_000

1. Bob calls [deposit()](https://github.com/sherlock-audit/2023-03-olympus/blob/main/sherlock-olympus/src/policies/BoostedLiquidity/BLVaultLido.sol#L143) by providing 10 wstETH where 1000 OHM is minted. Bob successfully stakes all of the 1000 OHM and 10 wstETH and proportionately receives 100 LP.

    OHM token balance: 101_000
    wstETH token balance: 1_010
    Total LP: 10_100
    Bob's LP: 100

3. Alice also calls [deposit()](https://github.com/sherlock-audit/2023-03-olympus/blob/main/sherlock-olympus/src/policies/BoostedLiquidity/BLVaultLido.sol#L143) by providing 10 wstETH with 1000 OHM. She successfully stakes all of the 1000 OHM and 10 wstETH and proportionately receives 100 LP too.

    OHM token balance: 102_000
    wstETH token balance: 1_020
    Total LP: 10_200
    Bob's LP: 100
    Alice's LP: 100

4. The pool has been well balanced at all time when suddenly there is a significant price drop in wstETH in the market (OHM: $15, wstETH: $1200). [`manager.getTknOhmPrice()`](https://github.com/sherlock-audit/2023-03-olympus/blob/main/sherlock-olympus/src/policies/BoostedLiquidity/BLVaultLido.sol#L232) is now 0.0125 (instead of the previous 0.01) wstETH per 1 OHM.

5. Bob exits the pool by removing all of his LP where he receives back 1000 OHM (burned) and 10 full wstETH since [expectedWstethAmountOut](https://github.com/sherlock-audit/2023-03-olympus/blob/main/sherlock-olympus/src/policies/BoostedLiquidity/BLVaultLido.sol#L233) is 1000 * 0.0125 = 12.5 (> 10).  

    OHM token balance: 101_000
    wstETH token balance: 1_010
    Total LP: 10_100
    Bob's LP: 0
    Alice's LP: 100

6. However, Alice, an avid staker, uses a flash loan to swap 100 wstETH for 9099.1 OHM to first make a profit of  9099.1 * 15 - 100 * 1200 = $16486.50. 

    OHM token balance: 91_900.9
    wstETH token balance: 1_110
    Total LP: 10_100
    Bob's LP: 0
    Alice's LP: 100

7. Next, Alice's LP is fully [withdrawn](https://github.com/sherlock-audit/2023-03-olympus/blob/main/sherlock-olympus/src/policies/BoostedLiquidity/BLVaultLido.sol#L203) where 909.91 OHM is received and [burned](https://github.com/sherlock-audit/2023-03-olympus/blob/main/sherlock-olympus/src/policies/BoostedLiquidity/BLVaultLido.sol#L244) while 10.99 wstETH is fully [transferred](https://github.com/sherlock-audit/2023-03-olympus/blob/main/sherlock-olympus/src/policies/BoostedLiquidity/BLVaultLido.sol#L247) to the user because [getTknOhmPrice()](https://github.com/sherlock-audit/2023-03-olympus/blob/main/sherlock-olympus/src/policies/BoostedLiquidity/BLVaultLido.sol#L232) is now 0.0125 wstETH per OHM, i.e. wstethAmountOut < expectedWstethAmountOut making [wstethToReturn == wstethAmountOut](https://github.com/sherlock-audit/2023-03-olympus/blob/main/sherlock-olympus/src/policies/BoostedLiquidity/BLVaultLido.sol#L236-L240). 

    OHM token balance: 90_990.99
    wstETH token balance: 1_099.01
    Total LP: 10_000
    Bob's LP: 0
    Alice's LP: 0

## Impact
Although Bob did not make any losses and has earned some rewards, Alice in contrast to Bob's action, has received an extra 10.99 - 10 = 0.99 wstETH all because the system can now be gamed without any penalty to the treasury while reaping a sizable flash swap profit of $16486.50 in step 5 .

## Code Snippet
[File: BLVaultLido.sol#L143-L200](https://github.com/sherlock-audit/2023-03-olympus/blob/main/sherlock-olympus/src/policies/BoostedLiquidity/BLVaultLido.sol#L143-L200)

[File: BLVaultLido.sol#L203-L256](https://github.com/sherlock-audit/2023-03-olympus/blob/main/sherlock-olympus/src/policies/BoostedLiquidity/BLVaultLido.sol#L203-L256)

## Tool used

Manual Review

## Recommendation
Consider restructuring the withdrawn ratios check if this is not intended to happen. For instance, an added check may be implemented checking whether or not the pool has been imbalanced. If it has been, the existing check should trigger an added slash commensurate with the amount of shift incurred. But this may not be the perfect fix since Alice could dodge it using two separate transactions instead of atomically.
