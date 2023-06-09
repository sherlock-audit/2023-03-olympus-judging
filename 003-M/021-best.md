hickuphh3

medium

# `claimRewards()` doesn't actually claim rewards

## Summary
`claimRewards()` calls `getReward()` with the owner as the address when it should be the vault itself (since the deposit is via the vault).

## Vulnerability Detail
```solidity
// Claim rewards from Aura
auraRewardPool().getReward(owner(), true);
```

Since deposits and withdrawals into the LP token is done via the vault, the staker would be the vault. Calling `getReward()` on `owner()` to claim rewards is therefore erroneous. It's likely to an accidental error since the other methods correctly calls on `address(this)`, eg.
```solidity
uint256 balRewards = auraRewardPool().earned(address(this));
```

Thankfully, rewards aren't locked up since anyone can directly call the aura reward pool to claim rewards for the vault, and the `withdrawAndUnwrap()` also correctly claims rewards upon withdrawals. It would merely be an inconvenience to the user to have to perform an additional transaction.

## POC
I performed integration testing (it took a while to sleuth out the required addresses, would've been nice if it was provided). Please take a look at `testClaimRewards()`.

```solidity
// SPDX-License-Identifier: Unlicense
pragma solidity 0.8.15;

import "forge-std/Test.sol";
import {FullMath} from "libraries/FullMath.sol";
import "../../../../src/policies/BoostedLiquidity/interfaces/IBalancer.sol";
import "../../../../src/policies/BoostedLiquidity/interfaces/IAura.sol";
import {IERC20} from "src/external/OlympusERC20.sol";
import "src/Kernel.sol";
import {MockPriceFeed} from "test/mocks/MockPriceFeed.sol";
import {OlympusMinter} from "modules/MINTR/OlympusMinter.sol";
import {OlympusTreasury} from "modules/TRSRY/OlympusTreasury.sol";
import {OlympusBoostedLiquidityRegistry} from "modules/BLREG/OlympusBoostedLiquidityRegistry.sol";
import {OlympusRoles, ROLESv1} from "modules/ROLES/OlympusRoles.sol";
import {RolesAdmin} from "policies/RolesAdmin.sol";
import {IBLVaultManagerLido, BLVaultManagerLido} from "policies/BoostedLiquidity/BLVaultManagerLido.sol";
import "policies/BoostedLiquidity/BLVaultLido.sol";

contract BLVaultIntegration is Test {
    // wstETH whale
    address user = 0x5fEC2f34D80ED82370F733043B6A536d7e9D7f8d;
    IERC20 liquidityPool = IERC20(0xd4f79CA0Ac83192693bce4699d0c10C66Aa6Cf0F);
    IERC20 aura = IERC20(0xC0c293ce456fF0ED870ADd98a0828Dd4d2903DBF);
    IERC20 bal = IERC20(0xba100000625a3754423978a60c9317c58a424e3D);
    IERC20 auraBAL = IERC20(0x616e8BfA43F920657B3497DBf40D6b1A02D4608d);
    IERC20 ohm = IERC20(0x64aa3364F17a4D01c6f1751Fd97C2BD3D7e7f1D5);
    IERC20 wsteth = IERC20(0x7f39C581F595B53c5cb19bD0b3f8dA6c935E2Ca0);
    IAuraMiningLib auraMiningLib = IAuraMiningLib(0x744Be650cea753de1e69BF6BAd3c98490A855f52);
    IAuraRewardPool auraRewardPool = IAuraRewardPool(0x636024F9Ddef77e625161b2cCF3A2adfbfAd3615);
    uint256 pid = 73;
    address auraBooster = 0xA57b8d98dAE62B26Ec3bcC4a365338157060B234;
    IVault balVault = IVault(0xBA12222222228d8Ba445958a75a0704d566BF2C8);
    IBalancerHelper balancerHelper = IBalancerHelper(0xE39B5e3B6D74016b2F6A9673D7d7493B6DF549d5);

    MockPriceFeed stethEthPriceFeed = MockPriceFeed(0x86392dC19c0b719886221c78AB11eb8Cf5c52812);
    MockPriceFeed ohmEthPriceFeed = MockPriceFeed(0x9a72298ae3886221820B1c878d12D872087D3a23);

    // Olympus DAO V3 contracts
    Kernel kernel = Kernel(0x2286d7f9639e8158FaD1169e76d1FbC38247f54b);
    address executor = kernel.executor();
    OlympusMinter minter = OlympusMinter(0xa90bFe53217da78D900749eb6Ef513ee5b6a491e);
    OlympusTreasury treasury = OlympusTreasury(0xa8687A15D4BE32CC8F0a8a7B9704a4C3993D9613);
    OlympusBoostedLiquidityRegistry blreg = new OlympusBoostedLiquidityRegistry(kernel);
    OlympusRoles rolesAdmin = OlympusRoles(0x6CAfd730Dc199Df73C16420C4fCAb18E3afbfA59);

    BLVaultManagerLido vaultManager;
    BLVaultLido vaultImplementation;
    BLVaultLido userVault;

    function setUp() public {
        vaultImplementation = new BLVaultLido();
        IBLVaultManagerLido.TokenData memory tokenData = IBLVaultManagerLido.TokenData({
            ohm: address(ohm),
            pairToken: address(wsteth),
            aura: address(aura),
            bal: address(bal)
        });

        IBLVaultManagerLido.BalancerData memory balancerData = IBLVaultManagerLido
            .BalancerData({
                vault: address(balVault),
                liquidityPool: address(liquidityPool),
                balancerHelper: address(balancerHelper)
            });
        
        IBLVaultManagerLido.AuraData memory auraData = IBLVaultManagerLido.AuraData({
            pid: pid,
            auraBooster: auraBooster,
            auraRewardPool: address(auraRewardPool)
        });

        IBLVaultManagerLido.OracleFeed memory ohmEthPriceFeedData = IBLVaultManagerLido
            .OracleFeed({feed: ohmEthPriceFeed, updateThreshold: uint48(1 days)});

        IBLVaultManagerLido.OracleFeed memory stethEthPriceFeedData = IBLVaultManagerLido
            .OracleFeed({feed: stethEthPriceFeed, updateThreshold: uint48(1 days)});
        
        vaultManager = new BLVaultManagerLido(
            kernel,
            tokenData,
            balancerData,
            auraData,
            address(auraMiningLib),
            ohmEthPriceFeedData,
            stethEthPriceFeedData,
            address(vaultImplementation),
            100_000e9,
            0
        );
        
        // Initialize system
        {
            vm.startPrank(executor);

            // Initialize modules
            kernel.executeAction(Actions.InstallModule, address(blreg));

            // Activate policies
            kernel.executeAction(Actions.ActivatePolicy, address(vaultManager));
            kernel.executeAction(Actions.ActivatePolicy, address(this));
            vm.stopPrank();
        }

         // grant this contract liquidity vault admin role
        {
            rolesAdmin.saveRole("liquidityvault_admin", address(this));
        }

        // Activate Vault Manager
        {
            vaultManager.activate();
        }

        // Prepare user's account
        // Create alice's vault
        vm.startPrank(user);
        userVault = BLVaultLido(vaultManager.deployVault());

        // Approve wstETH to alice's vault
        wsteth.approve(address(userVault), type(uint256).max);
        vm.stopPrank();
    }

    function testClaimRewards() public {
        vm.startPrank(user);
        userVault.deposit(10e18, 0);
        // move time forward
        skip(86400);
        // check rewards
        RewardsData[] memory rewardsData = userVault.getOutstandingRewards();

        assertGt(rewardsData[0].outstandingRewards, 0);
        assertGt(rewardsData[1].outstandingRewards, 0);

        uint256 balBeforeClaim = bal.balanceOf(user);
        uint256 auraBeforeClaim = aura.balanceOf(user);
        userVault.claimRewards();
        // user doesn't have his rewards claimed
        assertEq(bal.balanceOf(user) - balBeforeClaim, 0);
        assertEq(aura.balanceOf(user) - auraBeforeClaim, 0);
    }

    function configureDependencies() external returns (Keycode[] memory dependencies) {}

    function requestPermissions() external view returns (Permissions[] memory requests) {
       Keycode ROLES_KEYCODE = toKeycode("ROLES");

        requests = new Permissions[](2);
        requests[0] = Permissions(ROLES_KEYCODE, rolesAdmin.saveRole.selector);
        requests[1] = Permissions(ROLES_KEYCODE, rolesAdmin.removeRole.selector);
    }
}
```

## Impact
Rewards aren't claimed when expected to.

## Code Snippet
https://github.com/sherlock-audit/2023-03-olympus/blob/main/sherlock-olympus/src/policies/BoostedLiquidity/BLVaultLido.sol#L265

## Tool used
Manual Review, Fork testing with Foundry, buildbear.io for forking

## Recommendation
```diff
- auraRewardPool().getReward(owner(), true);
+ auraRewardPool().getReward();
```