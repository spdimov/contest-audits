Codebase: https://code4rena.com/reports/2024-07-munchables
Final Report: https://code4rena.com/reports/2024-07-munchables
# H-01: Inconsistent state of toiler's plotId, can lead to undeserved rewards for users and landlords
## Impact

Due to inconsistent state of the contract, users would be able to earn schnibbles for munchables they have staked on already missing plot. Respectively, landlords would earn schnibbles also for plots which they already dont own. This can either happen accidentally or done on purpose by malicious users/landlords.
## Proof of Concept
In `transferToUnoccupiedPlot()` function we update `toilerState[tokenId].latestTaxRate` but do not update the `toilerState[tokenId].plotId`

```solidity
function transferToUnoccupiedPlot(uint256 tokenId, uint256 plotId)
        external
        override
        forceFarmPlots(msg.sender)
        notPaused
    {
        (address mainAccount,) = _getMainAccountRequireRegistered(msg.sender);
        ToilerState memory _toiler = toilerState[tokenId];
        uint256 oldPlotId = _toiler.plotId;
        uint256 totalPlotsAvail = _getNumPlots(_toiler.landlord);
        if (_toiler.landlord == address(0)) revert NotStakedError();
        if (munchableOwner[tokenId] != mainAccount) revert InvalidOwnerError();
        if (plotOccupied[_toiler.landlord][plotId].occupied) {
            revert OccupiedPlotError(_toiler.landlord, plotId);
        }
        if (plotId >= totalPlotsAvail) revert PlotTooHighError();

        //@audit-issue plotId is not updated when transfering already staked munchable
        toilerState[tokenId].latestTaxRate = plotMetadata[_toiler.landlord].currentTaxRate;
        plotOccupied[_toiler.landlord][oldPlotId] = Plot({occupied: false, tokenId: 0});
        plotOccupied[_toiler.landlord][plotId] = Plot({occupied: true, tokenId: tokenId});

        emit FarmPlotLeave(_toiler.landlord, tokenId, oldPlotId);
        emit FarmPlotTaken(toilerState[tokenId], tokenId);
    }
```

which is later used to check if munchable is staked on a valid plot inside the `_farmPlots()` function

```solidity
function _farmPlots(address _sender) internal {
        (
            address mainAccount,
            MunchablesCommonLib.Player memory renterMetadata
        ) = _getMainAccountRequireRegistered(_sender);

        uint256[] memory staked = munchablesStaked[mainAccount];
        MunchablesCommonLib.NFTImmutableAttributes memory immutableAttributes;
        ToilerState memory _toiler;
        uint256 timestamp;
        address landlord;
        uint256 tokenId;
        int256 finalBonus;
        uint256 schnibblesTotal;
        uint256 schnibblesLandlord;
        for (uint8 i = 0; i < staked.length; i++) {
            timestamp = block.timestamp;
            tokenId = staked[i];
            _toiler = toilerState[tokenId];
            if (_toiler.dirty) continue;
            landlord = _toiler.landlord;
            // use last updated plot metadata time if the plot id doesn't fit
            // track a dirty bool to signify this was done once
            // the edge case where this doesnt work is if the user hasnt farmed in a while and the landlord
            // updates their plots multiple times. then the last updated time will be the last time they updated their plot details
            // instead of the first
            // @audit-issue toiler.plotId is not updated for transfered munchables
            if (_getNumPlots(landlord) < _toiler.plotId) {
                timestamp = plotMetadata[landlord].lastUpdated;
                toilerState[tokenId].dirty = true;
            }
            ...
    }

```

The following scenario could occur:
1. Landlord has 3 plots and have locked certain amount for them.
2. User stake on plot number 2.
3. User transfer from plot number 2 to plot number 3.
4. Landlord unlock part of his funds, leaving him with only 2 plots.
5. User and landlord continue to earn schnibbles.
## Tools Used

Manual review
## Recommended Mitigation Steps

In `transferToUnoccupiedPlot()` also update the `toilerState[tokenId].plotId` to keep the contract state consistent.

```solidity
function transferToUnoccupiedPlot(uint256 tokenId, uint256 plotId)
        external
        override
        forceFarmPlots(msg.sender)
        notPaused
    {
        (address mainAccount,) = _getMainAccountRequireRegistered(msg.sender);
        ToilerState memory _toiler = toilerState[tokenId];
        uint256 oldPlotId = _toiler.plotId;
        uint256 totalPlotsAvail = _getNumPlots(_toiler.landlord);
        if (_toiler.landlord == address(0)) revert NotStakedError();
        if (munchableOwner[tokenId] != mainAccount) revert InvalidOwnerError();
        if (plotOccupied[_toiler.landlord][plotId].occupied) {
            revert OccupiedPlotError(_toiler.landlord, plotId);
        }
        if (plotId >= totalPlotsAvail) revert PlotTooHighError();

        toilerState[tokenId].latestTaxRate = plotMetadata[_toiler.landlord].currentTaxRate;
+       toilerState[tokenId].plotId = plotId;
        plotOccupied[_toiler.landlord][oldPlotId] = Plot({occupied: false, tokenId: 0});
        plotOccupied[_toiler.landlord][plotId] = Plot({occupied: true, tokenId: tokenId});

        emit FarmPlotLeave(_toiler.landlord, tokenId, oldPlotId);
        emit FarmPlotTaken(toilerState[tokenId], tokenId);
    }
```

# M-01: Transaction ordering can result in user staking a munchible to higher tax rate than expected 

## Impact
Due to the unknown order of finalizing transactions and how they are going to be published to the L1 layer of Ethereum, users can stake their munchables to a much higher tax rate than expected. This leads to fewer rewards in schnibbles for the user and more for the landlord.

## Proof of Concept

`stakeMunchable()` function does not provide any protection mechanisms for the tax rate on the plot where users are going to stake their munchable.

```solidity
function stakeMunchable(
        address landlord,
        uint256 tokenId,
        uint256 plotId
    ) external override forceFarmPlots(msg.sender) notPaused {
	...
}
```

Users might expect to stake their munchable on a plot with tax rate = X, but after they have submitted their transaction, a landlord's transaction updating the tax rate to Y (Y being much higher than X) might be published before the user's one.

```solidity
function updateTaxRate(uint256 newTaxRate) external override notPaused {
        ...
    }
```
## Tools Used

Manual review
## Recommended Mitigation Steps

Consider adding one more parameter to the `stakeMunchable()`function which states the expected tax rate. Check could be strict or to allow small difference between expected and actual.

```solidity
function stakeMunchable(
        address landlord,
        uint256 tokenId,
        uint256 plotId,
        expectedTaxRate
    ) external override forceFarmPlots(msg.sender) notPaused {
	(address mainAccount, ) = _getMainAccountRequireRegistered(msg.sender);
        if (landlord == mainAccount) revert CantStakeToSelfError();       
+	if(plotMetadata[landlord].currentTaxRate != expectedTaxRate) {
+           revert TaxRateDoesNotMatchUserExpectations();
+       }
        ...
}
```



# M-02: All user\`s staked munchables will be temporary stuck if they have munchable staked on invalid plot and tax rate is not updated

## Impact
Users wont be able to farm and respectively wont be able to stake/unstake, since farming is done before both, due to underflow issue when calculating schnibbles rewards.
This happens in the following case:
1. Tax rate is updated
2. User stake munchable on plot X.
3. Landlord unlock funds, thus making plot X invalid for receiving rewards.
4. User tries to do any of the farm/stake/unstake activities.

This state can be exited by calling `updatePlotMetadata()` which will update the plot metadata and fix the underflow issue but since this function that can only be called by AccountManager contract, all munchables will be stuck until this point.
## Proof of Concept

When farming, in order to calculate the rewards for users, `_farmPlots()` function goes through all the staked munchable and for each it calculated the time for which a munchable has been staked. 
For VALID plots it is calculated by getting the difference between current `block.timestamp` and `_toiler.lastToilDate`.
For INVALID plots it is calculated by getting the difference between current `plotMetadata[landlord].lastUpdated` and `_toiler.lastToilDate`.

```solidity
function _farmPlots(address _sender) internal {
        (
            address mainAccount,
            MunchablesCommonLib.Player memory renterMetadata
        ) = _getMainAccountRequireRegistered(_sender);

        uint256[] memory staked = munchablesStaked[mainAccount];
        MunchablesCommonLib.NFTImmutableAttributes memory immutableAttributes;
        ToilerState memory _toiler;
        uint256 timestamp;
        address landlord;
        uint256 tokenId;
        int256 finalBonus;
        uint256 schnibblesTotal;
        uint256 schnibblesLandlord;
        for (uint8 i = 0; i < staked.length; i++) {
            timestamp = block.timestamp; // "happy path" uses current timestamp to calculate staked time
            tokenId = staked[i];
            _toiler = toilerState[tokenId];
            if (_toiler.dirty) continue;
            landlord = _toiler.landlord;
            // use last updated plot metadata time if the plot id doesn't fit
            // track a dirty bool to signify this was done once
            // the edge case where this doesnt work is if the user hasnt farmed in a while and the landlord
            // updates their plots multiple times. then the last updated time will be the last time they updated their plot details
            // instead of the first
            if (_getNumPlots(landlord) < _toiler.plotId) {
                timestamp = plotMetadata[landlord].lastUpdated; // time of the last tax rate update is used for invalid plot
                toilerState[tokenId].dirty = true;
            }
            (
                ,
                MunchablesCommonLib.Player memory landlordMetadata
            ) = _getMainAccountRequireRegistered(landlord);

            immutableAttributes = nftAttributesManager.getImmutableAttributes(
                tokenId
            );
            finalBonus =
                int16(
                    REALM_BONUSES[
                        (uint256(immutableAttributes.realm) * 5) +
                            uint256(landlordMetadata.snuggeryRealm)
                    ]
                ) +
                int16(
                    int8(RARITY_BONUSES[uint256(immutableAttributes.rarity)])
                );
            schnibblesTotal =
                (timestamp - _toiler.lastToilDate) * // time staked - okay for "happy path", could result in underflow for certain conditions
                BASE_SCHNIBBLE_RATE;
            schnibblesTotal = uint256(
                (int256(schnibblesTotal) +
                    (int256(schnibblesTotal) * finalBonus)) / 100
            );
            schnibblesLandlord =
                (schnibblesTotal * _toiler.latestTaxRate) /
                1e18;
```

Consider the following case:
1. Tax rate is updated at timestamp X.  `plotMetadata[landlord].lastUpdated` = X
2. User stake munchables at timestamp X + n.  `_toiler.lastToilDate` = X + n
3. Calculating `plotMetadata[landlord].lastUpdated`- `_toiler.lastToilDate` would result in underflow issue which will revert the transaction
## Tools Used

Manual review
## Recommended Mitigation Steps

"Happy path" calculation can be used in this case -  `block.timestamp` - ` _toiler.lastToilDate`
This way will even calculate the rewards more fairly than waiting for updating `plotMetadata[landlord].lastUpdated` as users would get smaller amount of rewards for their munchable on invalid plot.

```solidity
...
if (_getNumPlots(landlord) < _toiler.plotId) {
    timestamp = plotMetadata[landlord].lastUpdated > _toiler.lastToilDate
        ? plotMetadata[landlord].lastUpdated
        : block.timestamp;
    toilerState[tokenId].dirty = true;
}
```

# L-01: uint to int cast may overflow in _farmPlots() function.

## Impact

`uint256` to `int256` cast may result in overflow for schnibblesTotal values between 2^255 and 2^256 - 1 . This depends on the time of which a munchable has been stake and the `BASE_SCHNIBBLE_RATE`. This would lead to complete stuck state for the farming/staking/unstaking activities as all functions will make the transactions revert.
 
```solidity
schnibblesTotal = (timestamp - _toiler.lastToilDate) *
	                BASE_SCHNIBBLE_RATE;
schnibblesTotal = uint256(
                (int256(schnibblesTotal) +
                    (int256(schnibblesTotal) * finalBonus)) / 100
            );
```

## Mitigation

Keep in mind the possibility for this overflow when setting BASE_SCHNIBBLE.

# L-02: Users wont be able to unstake munchable when contract is in paused stake

## Impact

Users wont be able unstake munchable NFTs out of the contract if it is paused state. If those NFTs are considered to have financial value, users may want to be able to withdraw them at any time.

```solidity
function unstakeMunchable(uint256 tokenId) external override forceFarmPlots(msg.sender) notPaused {
	...
}
```

## Mitigation

Remove `notPaused` modifier and allow users to withdraw their NFTs.






# Missed Findings 

## H-02: Invalid validation in `_farmPlots` function allowing a malicious user repeated farming without locked funds

```solidity
if (_getNumPlots(landlord) < _toiler.plotId) { timestamp = plotMetadata[landlord].lastUpdated; toilerState[tokenId].dirty = true; }
```

Lesson Learned: Always verify if the comparison is correct (edge cases, correct sign used, < instead of <=)


## H-03: Miscalculation in `_farmPlots` function could lead to a user unable to unstake all NFTs

```solidity
finalBonus =
                int16(
                    REALM_BONUSES[
                        (uint256(immutableAttributes.realm) * 5) +
                            uint256(landlordMetadata.snuggeryRealm)
                    ]
                ) +
                int16(
                    int8(RARITY_BONUSES[uint256(immutableAttributes.rarity)])
                );
```

```solidity
schnibblesTotal = uint256(
                (int256(schnibblesTotal) +
                    (int256(schnibblesTotal) * finalBonus)) / 100
            );
```

Lesson Learned: int to uint conversion can lead to overflow, verify that it is not possible



## H-05 Failure to update dirty flag in `transferToUnoccupiedPlot` prevents reward accumulation on valid plot

Basic workflow state machine checks should have been enough here.


## M-01 Users can farm on zero-tax land if the landlord locked tokens before the LandManager deployment

Verifiy that users cannot do harm after contract is deployed but not initialized
