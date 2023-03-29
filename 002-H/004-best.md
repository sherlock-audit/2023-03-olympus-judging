0x52

high

# Adversary can stake LP directly for the vault then withdraw to break lp accounting in BLVaultManagerLido

## Summary

The AuraRewardPool allows users to stake directly for other users. In this case the malicious user could stake LP directly for their vault then call withdraw on their vault. This would cause the LP tracking to break on BLVaultManagerLido. The result is that some users would now be permanently trapped because their vault would revert when trying to withdraw.

## Vulnerability Detail

[BaseRewardPool.sol#L196-L207](https://github.com/aurafinance/convex-platform/blob/1d6e9c403a4440c712396422e1bd5af7e5ea1ecf/contracts/contracts/BaseRewardPool.sol#L196-L207)

    function stakeFor(address _for, uint256 _amount)
        public
        returns(bool)
    {
        _processStake(_amount, _for);

        //take away from sender
        stakingToken.safeTransferFrom(msg.sender, address(this), _amount);
        emit Staked(_for, _amount);
        
        return true;
    }

AuraRewardPool allows users to stake directly for another address with them receiving the staked tokens.

[BLVaultLido.sol#L218-L224](https://github.com/sherlock-audit/2023-03-olympus/blob/main/sherlock-olympus/src/policies/BoostedLiquidity/BLVaultLido.sol#L218-L224)

        manager.decreaseTotalLp(lpAmount_);

        // Unstake from Aura
        auraRewardPool().withdrawAndUnwrap(lpAmount_, claim_);

        // Exit Balancer pool
        _exitBalancerPool(lpAmount_, minTokenAmounts_);

Once the LP has been stake the adversary can immediately withdraw it from their vault. This calls decreaseTotalLP on BLVaultManagerLido which now permanently break the LP account.

[BLVaultManagerLido.sol#L277-L280](https://github.com/sherlock-audit/2023-03-olympus/blob/main/sherlock-olympus/src/policies/BoostedLiquidity/BLVaultManagerLido.sol#L277-L280)

    function decreaseTotalLp(uint256 amount_) external override onlyWhileActive onlyVault {
        if (amount_ > totalLp) revert BLManagerLido_InvalidLpAmount();
        totalLp -= amount_;
    }
    
If the amount_ is ever greater than totalLP it will cause decreaseTotalLP to revert. By withdrawing LP that was never deposited to a vault, it permanently breaks other users from being able to withdraw.

Example:
User A deposits wstETH to their vault which yields 50 LP. User B creates a vault then stake 50 LP and withdraws it from his vault. The manager now thinks there is 0 LP in vaults. When User A tries to withdraw their LP it will revert when it calls manger.decreaseTotalLp. User A is now permanently trapped in the vault.

## Impact

LP accounting is broken and users are permanently trapped.

## Code Snippet

[BLVaultLido.sol#L203-L256](https://github.com/sherlock-audit/2023-03-olympus/blob/main/sherlock-olympus/src/policies/BoostedLiquidity/BLVaultLido.sol#L203-L256)

## Tool used

Manual Review

## Recommendation

Individual vaults should track how much they have deposited and shouldn't be allowed to withdraw more than deposited.