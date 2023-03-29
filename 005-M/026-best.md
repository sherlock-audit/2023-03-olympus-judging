ABA

medium

# `getOhmSupplyChangeData` shows faulty data, will mislead external integrators/users

## Summary

External sources will use the `getOhmSupplyChangeData`, to display information to user about the state of the system.
This function always returns 0 for the deployed OHM values and circulating burned OHM.

## Vulnerability Detail

`getOhmSupplyChangeData` from `BLVaultManagerLido` aggregates and returns 3 pieces of information:

https://github.com/sherlock-audit/2023-03-olympus/blob/main/sherlock-olympus/src/policies/BoostedLiquidity/BLVaultManagerLido.sol#L424-L437
```Solidity
    function getOhmSupplyChangeData()
        external
        view
        override
        returns (uint256 poolOhmShare, uint256 deployedOhm, uint256 circulatingOhmBurned)
    {
        // Net emitted is the amount of OHM that was minted to the pool but is no longer in the
        // pool beyond what has been burned in the past. Net removed is the amount of OHM that is
        // in the pool but wasn’t minted there plus what has been burned in the past. Here we just return
        // the data components to calculate that.


        uint256 currentPoolOhmShare = getPoolOhmShare();
        return (currentPoolOhmShare, deployedOhm, circulatingOhmBurned);
    }
```

out of these `deployedOhm` and `circulatingOhmBurned` will always be 0 because they are both declared as returns variables:

https://github.com/sherlock-audit/2023-03-olympus/blob/main/sherlock-olympus/src/policies/BoostedLiquidity/BLVaultManagerLido.sol#L428
```Solidity
        returns (uint256 poolOhmShare, uint256 deployedOhm, uint256 circulatingOhmBurned)
```

and as contract variables

https://github.com/sherlock-audit/2023-03-olympus/blob/main/sherlock-olympus/src/policies/BoostedLiquidity/BLVaultManagerLido.sol#L77-L78
```Solidity
    uint256 public deployedOhm;
    uint256 public circulatingOhmBurned;
```

As the local variables shadow the global ones, the return values will always be 0.

## Impact

Users will be mislead about the current state of the system. Also, any decision the team will base off this information (e.g. modifying deployed OHM limit) will be erroneous.
Also, any other integration with this contract via this function will receive faulty information.

## Code Snippet

A simple POC to be set in `BLVaultManagerLidoMocks.t` to highlight that even if vault minted OHM, it is not reflected by `getOhmSupplyChangeData`
```Solidity
    /// [X]  getOhmSupplyChangeData
    ///     [X]  returns incorrect amounts

    function testIncorrectness_getOhmSupplyChangeData() public {
        address aliceVault = _createVault();

        // Increase OHM minted, this also increases the deployedOhm 
        vm.prank(aliceVault);
        vaultManager.mintOhmToVault(99_900e9);

        (uint256 poolOhmShare, uint256 deployedOhm, uint256 circulatingOhmBurned) = vaultManager.getOhmSupplyChangeData();
        
        // show the value has not changed
        assertEq(poolOhmShare, 0);
        assertEq(deployedOhm, 0);
        assertEq(circulatingOhmBurned, 0);
    }
```

## Tool used

Manual Review
Visual Studio Code with the `Solidity Visual Developer` plugin. It highlighted the issue
https://marketplace.visualstudio.com/items?itemName=tintinweb.solidity-visual-auditor

## Recommendation

Remove the return variables on line:
https://github.com/sherlock-audit/2023-03-olympus/blob/main/sherlock-olympus/src/policies/BoostedLiquidity/BLVaultManagerLido.sol#L428