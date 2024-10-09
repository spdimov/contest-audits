# H-1: Users can bid/unbid after auction has ended leading to unfair distribution of auction tokens and DoS

## Summary

In the `FjordAuction` contract, the auction has a defined end time, stored in the state variable `auctionEndTime`. Users are allowed to place and withdraw bids until `block.timestamp` reaches this specified end time. Once this point is reached, anyone can call the `auctionEnd()` function to finalize the auction. After the auction is finalized, users can claim their auction tokens based on their bids.

However, the current implementation has a vulnerability: users can call `auctionEnd()` to end the auction and then place additional bids afterwards.

## Vulnerability Details

[https://github.com/Cyfrin/2024-08-fjord/blob/0312fa9dca29fa7ed9fc432fdcd05545b736575d/src/FjordAuction.sol#L143](https://github.com/Cyfrin/2024-08-fjord/blob/0312fa9dca29fa7ed9fc432fdcd05545b736575d/src/FjordAuction.sol#L143) [https://github.com/Cyfrin/2024-08-fjord/blob/0312fa9dca29fa7ed9fc432fdcd05545b736575d/src/FjordAuction.sol#L159](https://github.com/Cyfrin/2024-08-fjord/blob/0312fa9dca29fa7ed9fc432fdcd05545b736575d/src/FjordAuction.sol#L159) [https://github.com/Cyfrin/2024-08-fjord/blob/0312fa9dca29fa7ed9fc432fdcd05545b736575d/src/FjordAuction.sol#L181](https://github.com/Cyfrin/2024-08-fjord/blob/0312fa9dca29fa7ed9fc432fdcd05545b736575d/src/FjordAuction.sol#L181)

`bid()` and `auctionEnd()` have the corresponding checks to see if auction has ended based on the current `block.timestamp`

```
    function bid(uint256 amount) external {
        if (block.timestamp > auctionEndTime) {
            revert AuctionAlreadyEnded();
        }

        bids[msg.sender] = bids[msg.sender].add(amount);
        totalBids = totalBids.add(amount);

        fjordPoints.transferFrom(msg.sender, address(this), amount);
        emit BidAdded(msg.sender, amount);
    }
```

```
    function unbid(uint256 amount) external {
        if (block.timestamp > auctionEndTime) {
            revert AuctionAlreadyEnded();
        }

        uint256 userBids = bids[msg.sender];
        if (userBids == 0) {
            revert NoBidsToWithdraw();
        }
        if (amount > userBids) {
            revert InvalidUnbidAmount();
        }

        bids[msg.sender] = bids[msg.sender].sub(amount);
        totalBids = totalBids.sub(amount);
        fjordPoints.transfer(msg.sender, amount);
        emit BidWithdrawn(msg.sender, amount);
    }
```

```
    function auctionEnd() external {
        if (block.timestamp < auctionEndTime) {
            revert AuctionNotYetEnded();
        }
        if (ended) {
            revert AuctionEndAlreadyCalled();
        }

        ended = true;
        emit AuctionEnded(totalBids, totalTokens);

        if (totalBids == 0) {
            auctionToken.transfer(owner, totalTokens);
            return;
        }

        multiplier = totalTokens.mul(PRECISION_18).div(totalBids);

        // Burn the FjordPoints held by the contract
        uint256 pointsToBurn = fjordPoints.balanceOf(address(this));
        fjordPoints.burn(pointsToBurn);
    }
```

Neither the `bid()` nor the `unbid()`, nor `auctionEnd()` functions explicitly check if the current `block.timestamp` is exactly equal to `auctionEndTime`. Since the `block.timestamp` is set by the block miner, it can be adjusted to match the `auctionEndTime`.

Malicious user could exploit the situation by executing `auctionEnd()` followed by `bid()` , `claimTokens()` and `unbid()` in the same transaction. During the `auctionEnd()` call, a multiplier is calculated based on the total bids at that moment. This multiplier is later used to determine the claimable rewards.

This scenario leads to two issues:

1. The malicious user can obtain a more favorable ratio for their Fjord Points by bidding after `auctionEnd()` has been called, thereby manipulating the calculated multiplier. This results in an unfair distribution of rewards. Additionally, the malicious user can then withdraw his fjord points by `unbid()`.
    
2. After the malicious user claims their auction tokens, the contract may not have enough tokens left to distribute to other legitimate bidders. This imbalance could cause a DoS situation, as other bidders would be unable to claim their rewards until the contract is replenished with additional tokens by the admin.
    

## Impact

As described in the Vulnerability Details section, the impact is unfair distribution of auction tokens and limited DoS.

## Tools Used

Manual Review.

## Recommendations

Update the checks inside `bid()` and `unbid()` functions to also check the `ended` variable.

```
    function bid(uint256 amount) external {
-       if (block.timestamp > auctionEndTime) {
+       if (block.timestamp > auctionEndTime || ended) {
            revert AuctionAlreadyEnded();
        }

        bids[msg.sender] = bids[msg.sender].add(amount);
        totalBids = totalBids.add(amount);

        fjordPoints.transferFrom(msg.sender, address(this), amount);
        emit BidAdded(msg.sender, amount);
    }
```

```
    function unbid(uint256 amount) external {
-       if (block.timestamp > auctionEndTime) {
+       if (block.timestamp > auctionEndTime || ended) {
            revert AuctionAlreadyEnded();
        }

        uint256 userBids = bids[msg.sender];
        if (userBids == 0) {
            revert NoBidsToWithdraw();
        }
        if (amount > userBids) {
            revert InvalidUnbidAmount();
        }

        bids[msg.sender] = bids[msg.sender].sub(amount);
        totalBids = totalBids.sub(amount);
        fjordPoints.transfer(msg.sender, amount);
        emit BidWithdrawn(msg.sender, amount);
    }
```

# M-1: Epoch can be left without rewards if addRewards is executed at the end of the epoch

## Summary

An issue arises if the `addRewards()` function in the `FjordStaking` contract is called near the end of an epoch but does not get executed before the epoch ends. This situation could result in the epoch ending without any rewards being distributed, leaving users without the expected rewards for that period.

Although the function's NatSpec documentation indicates that `addRewards()` is intended to be called at the end of an epoch to trigger the epoch rollover, this reliance on precise timing creates a risk that rewards may not be added in time, potentially impacting the reward distribution process.

## Vulnerability Details

If the Ethereum network experiences high congestion near the end of an epoch, transactions may take longer to confirm. In such cases, the `addRewards()` transaction might not be included in a block before the epoch ends, resulting in no rewards being distributed for that epoch.
Another scenario might be a malicious user engaging in block stuffing—filling blocks with their transactions to delay or prevent the inclusion of the `addRewards()` transaction. By doing so, the attacker can manipulate the timing to cause the epoch to end without rewards being added, potentially disrupting the reward distribution mechanism.
## Impact

The primary impact of this vulnerability is the inability to distribute rewards, which disrupts the core functionality of the protocol. Since reward distribution is fundamental to incentivizing user participation and maintaining trust in the protocol, any failure to distribute rewards undermines the protocol's reliability and user confidence. Given the essential nature of this functionality, the impact is considered high. The likelihood of this issue occurring is low/medium as this does not have direct incentives for the attacker
## Tools Used

Manual review.

## Recommendations

To mitigate this issue, avoid calling `addRewards()` near the end of an epoch. Another design would be to implement predefined minimum rewards for each epoch that are transferred in advance, reducing reliance on the timing of `addRewards()`. This approach ensures that each epoch has guaranteed rewards, regardless of when additional rewards are added.

