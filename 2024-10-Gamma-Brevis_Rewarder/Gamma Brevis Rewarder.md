## Users cannot claim rewards for different epochs for the same distribution

### Summary

Users submit claims to Brevis for rewards based on their staking period in Gamma vaults. Brevis verifies staking duration, then triggers `GammaRewarder`'s callback. Incentivisors set distribution periods, split into epochs, with rewards allocated per epoch. Claiming rewards after each epoch that has passed and users are eligible for rewards wont work after the first claim because of the current implementation.
### Root Cause
In `handleProofResult()` it is checked that the user does not have previous checks for the distribution in context. 
In case there are multiple epoch from which users can claim rewards, it wont be possible to claim rewards for them separately and sequentially after the first epoch since after the first claim, `claim.amount` wont be zero and for all subsequent tries it will hold the value of the rewards from the first claimed epoch as it would be the same distribution. 

```solidity
    function handleProofResult(bytes32, bytes32 _vkHash, bytes calldata _appCircuitOutput) internal override {
        ...
        
        CumulativeClaim memory claim = claimed[userAddress][rewardTokenAddress][distributionId];
        require(claim.amount == 0 , "Already claimed reward.");

        claim.startBlock = startBlock;
        claim.endBlock = endBlock;
        claim.amount = totalRewardAmount;
        claimed[userAddress][rewardTokenAddress][distributionId] = claim;

		...
    }
```
https://github.com/sherlock-audit/2024-10-gamma-rewarder/blob/475f7fbd0f7c2717ed585a67632e9a675b51c306/GammaRewarder/contracts/GammaRewarder.sol#L209

### Internal pre-conditions

Distribution with epochs more than 1.

### External pre-conditions

User claim rewards after the first epoch has finished.

### Attack Path

Consider a distribution split into 2 epochs, where staking in each epoch entitles users to a part of reward X. If a user stakes throughout both epochs, they earn part of  2 * X.

After the first epoch, Alice is eligible for a partial reward based on her staking during this period. She submits her claim through Brevis, which verifies her eligibility. The claim proceeds successfully in the callback function of `GammaRewarder` because `claimed[userAddress][rewardTokenAddress][distributionId]` is initially empty, and `claim.amount == 0` is true.

Upon claiming, the following updates occur:
- `claim.startBlock = startBlock;`
- `claim.endBlock = endBlock;`
- `claim.amount = totalRewardAmount;`
These updated claim properties are then stored in `claimed[userAddress][rewardTokenAddress][distributionId]`.

After the next epoch, Alice tries to claim her rewards again. However, `claimed[userAddress][rewardTokenAddress][distributionId].amount` is now non-zero, so the claim is blocked, even though she is eligible and has not claimed her rewards for this epoch yet.

### Impact

DoS of a core functionality with lock of user funds.

### PoC

N/A

### Mitigation

To improve tracking, update `claimed` to store the last claimed epoch per distribution. This approach would assume that users will sequentially claim rewards - it wont be possible to claim for epoch X + 1 and the claim for epoch X.
Another approach would be to keep track of the claimed epochs for each distribution and verify that claim for epoch has not happened already.

Additionally tracking the `rewardTokenAddress` in this case is not needed as `distributionId` already uniquely identifies the distribution and token

