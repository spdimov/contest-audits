# H-1: Locked ETH is not updated upon players refund, resulting in permanent stuck funds

## Summary

Raffles that fail to meet the minimum ticket threshold can be cancelled, and users are refunded. However, incorrect accounting of locked ETH after refunds results in permanently stuck funds within the contract.
## Vulnerability Detail

`refundPlayers()` function is expected to distribute back the price players have paid to participate in already cancelled raffle.

```solidity
function refundPlayers(uint256 raffleId, address[] calldata players) external {
        Raffle storage raffle = _raffles[raffleId];
        if (raffle.status != RaffleStatus.CANCELED) revert InvalidRaffle();
        for (uint256 i = 0; i < players.length; ) {
            address player = players[i];
            uint256 participation = uint256(raffle.participations[player]);
            if (((participation >> 160) & 1) == 1) revert PlayerAlreadyRefunded(player);
            raffle.participations[player] = bytes32(participation | (1 << 160));
            uint256 amountToSend = (participation & type(uint128).max);
            _sendETH(amountToSend, player);
            emit PlayerRefund(raffleId, player, bytes32(participation));
            unchecked { ++i; }
        }
    }
```

The `_lockedETH` state variable tracks the ETH received from players purchasing tickets and is reduced when a raffle is won. Its purpose is to determine how much ETH the Admin can withdraw from the contract, ensuring that only ETH from completed raffles is withdrawable.

```solidity
function withdrawETH() external onlyRole(0) {
        uint256 balance;
        unchecked {
            balance = address(this).balance - _lockedETH;
        }
        _sendETH(balance, msg.sender);
    }

```

In the current implementation, the `refundPlayers()` function does not update the `_lockedFunds` variable after refunding players, resulting in an inconsistent contract state.
`_lockedFunds` will always increase and will not be consistent with the actual balance of the contract.

The withdrawal amount is calculated using the expression `unchecked {balance = address(this).balance - _lockedETH;}`. If the `_lockedETH` exceeds the contract's current balance (which will happen after the first refund) this subtraction will cause an underflow, leading to `balance` being set to a very large number. As a result, the contract will attempt to transfer more ETH than it actually holds, which will cause the transaction to fail and prevent any withdrawals. 
## Impact

Permanent stuck funds.
## Code Snippet

https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/81b28633d0f450e33a8b32976e17122418f5d47e/public-contracts/contracts/WinnablesTicketManager.sol#L215

https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/81b28633d0f450e33a8b32976e17122418f5d47e/public-contracts/contracts/WinnablesTicketManager.sol#L300
## Tool used

Manual Review

## Recommendation

Consider substracting the `_lockedEth` variable in `refundPlayers()` function:

```diff solidity
function refundPlayers(uint256 raffleId, address[] calldata players) external {
        Raffle storage raffle = _raffles[raffleId];
        if (raffle.status != RaffleStatus.CANCELED) revert InvalidRaffle();
        for (uint256 i = 0; i < players.length; ) {
            address player = players[i];
            uint256 participation = uint256(raffle.participations[player]);
            if (((participation >> 160) & 1) == 1) revert PlayerAlreadyRefunded(player);
            raffle.participations[player] = bytes32(participation | (1 << 160));
            uint256 amountToSend = (participation & type(uint128).max);
+           _lockedETH -= amountToSend;
            _sendETH(amountToSend, player);
            emit PlayerRefund(raffleId, player, bytes32(participation));
            unchecked { ++i; }
        }
    }
```


# H-2: Admin can prevent winners from withdrawing his prize

## Summary

The README of the contest clearly states that "Winnables admins cannot do anything to prevent a winner from withdrawing their prize." However, in the current implementation, this invariant can be violated. The admin has the ability to prevent winners from withdrawing their prize, which contradicts the intended design and could undermine the fairness of the protocol.
## Vulnerability Detail

The issue stems from the ability of admin to override CCIP counterpart before `propagateRaffleWinner()` is called

```solidity
function propagateRaffleWinner(address prizeManager, uint64 chainSelector, uint256 raffleId) external {
        Raffle storage raffle = _raffles[raffleId];
        if (raffle.status != RaffleStatus.FULFILLED) revert InvalidRaffleStatus();
        raffle.status = RaffleStatus.PROPAGATED;
        address winner = _getWinnerByRequestId(raffle.chainlinkRequestId);

        _sendCCIPMessage(prizeManager, chainSelector, abi.encodePacked(uint8(CCIPMessageType.WINNER_DRAWN), raffleId, winner));
        IWinnablesTicket(TICKETS_CONTRACT).refreshMetadata(raffleId);
        unchecked {
            _lockedETH -= raffle.totalRaised;
        }
    }
```

Initially, the admin can set the CCIP counterpart correctly, for example, by linking the `WinnablesPrizeManager` contract on the Ethereum mainnet:
```solidity
ticketManager.setCCIPCounterpart(address(prizeManager), 5009297550715157269, true);   
```

However, the admin can later override the `enabled` status of this counterpart:
```solidity
ticketManager.setCCIPCounterpart(address(prizeManager), 5009297550715157269, false);
```

This would prevent the contracts from communicating with each other, effectively locking the winner's funds and disrupting the entire functionality of the system.
## Impact

Admin can prevent winner from withdrawing his prize and DoS of the whole system.
## Code Snippet

https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/81b28633d0f450e33a8b32976e17122418f5d47e/public-contracts/contracts/WinnablesTicketManager.sol#L238
https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/81b28633d0f450e33a8b32976e17122418f5d47e/public-contracts/contracts/WinnablesTicketManager.sol#L334
## Tool used

Manual Review
## Recommendation

Consider removing the possibility to disable a counterpart or implement additional logic for special cases where disabling should be supported

# M-1: Perpetual frontrunning of raffle creation leading to inability to create raffles and CCIP fee token loss

## Summary

Creating a raffle is a two-step process. First, the Admin locks the prize (NFTs, Tokens, or ETH) in the `WinnablesPrizeManager` contract, which then sends a CCIP message to the `WinnablesTicketManager`, incurring a cost in tokens. After this, the Admin creates the raffle in the `WinnablesTicketManager` contract. A raffle can be canceled if it reaches its end time without surpassing the minimum ticket threshold, and this cancellation can be initiated by anyone.

However, there's a potential issue: if the Admin locks a prize but hasn't yet created the raffle, users can exploit this by frontrunning the `createRaffle` transaction, effectively preventing the Admin from creating the raffle. Additionally, when a raffle is canceled, the `WinnablesTicketManager` sends a CCIP message back to the `WinnablesPrizeManager`, further incurring costs in tokens. This exploit could lead to unnecessary expenses and disrupt the raffle creation process.
## Vulnerability Detail

The `cancelRaffle()` function in `WinnablesTicketManager.sol` is designed to cancel a raffle in two scenarios:
*  When the raffle has finished but did not meet the required ticket threshold.
* When the prize has been locked in the `WinnablesPrizeManager` contract, but the raffle has not yet been created.

This function can be called by anyone, and there is no verification of the caller in either scenario. The second scenario would prevent the Admin to successfully create a raffle because of the check on line `#264`
`if (raffle.status != RaffleStatus.PRIZE_LOCKED) revert PrizeNotLocked();` as status would be already `CANCELED`.

```solidity
function cancelRaffle(address prizeManager, uint64 chainSelector, uint256 raffleId) external {
        _checkShouldCancel(raffleId);

        _raffles[raffleId].status = RaffleStatus.CANCELED;
        _sendCCIPMessage(
            prizeManager,
            chainSelector,
            abi.encodePacked(uint8(CCIPMessageType.RAFFLE_CANCELED), raffleId)
        );
        IWinnablesTicket(TICKETS_CONTRACT).refreshMetadata(raffleId);
    }
```

```solidity
function _checkShouldCancel(uint256 raffleId) internal view {
        Raffle storage raffle = _raffles[raffleId];
        if (raffle.status == RaffleStatus.PRIZE_LOCKED) return;
        if (raffle.status != RaffleStatus.IDLE) revert InvalidRaffle();
        if (raffle.endsAt > block.timestamp) revert RaffleIsStillOpen();
        uint256 supply = IWinnablesTicket(TICKETS_CONTRACT).supplyOf(raffleId);
        if (supply > raffle.minTicketsThreshold) revert TargetTicketsReached();
    }
```

While the described scenario is indeed a DoS of a single block, it's important to highlight that each operation—specifically the combination of locking a prize and then canceling the raffle—incurs message fees due to the CCIP messages sent between the `WinnablesPrizeManager` and `WinnablesTicketManager` contracts. Although the DoS impact is limited to a single block, the repeated triggering of this process can lead to a significant cumulative loss of tokens for the protocol

Lets examine the billing mechanism for CCIP Messages to get an idea of the protocol losses by each frontrunned transaction.
Source: https://docs.chain.link/ccip/billing

The total fee for sending a CCIP message is calculated as follows:
`fee = blockchain fee + network fee`

For our example, the blockchain fee consists solely of the execution cost, as data availability cost is zero since both chains involved are Layer 1 (L1). The execution cost is determined by:

`blockchain fee = execution cost + data availability cost`

`execution cost = gas price * gas usage * gas multiplier`

`gas usage = gas limit + destination gas overhead` + `destination gas per payload` + `gas for token transfers`

In this case, let's simplify by assuming that the destination gas overhead, destination gas per payload, gas for token transfers, and gas multiplier are all zero, leaving us with:

`gas usage = gas limit = 200 000 (default value)`

Given the average gas price on Ethereum at the time of writing is 5 gwei:

At the time of writing the report average gas price in Ethereum is `5 gwei` (gas price is calculated on the destination chain)

`execution cost` = `200 000` * `5 gwei` =`1 000 000 gwei`
which makes `0.001 ETH` ~ `2.60$` (with current Ether price = $2650)
Gas prices can fluctuate, potentially doubling or tripling during periods of high network congestion.

With the network fee fixed at $0.50, the total message cost is approximately $3.

The cost of sending a message from Ethereum to Avalanche is approximately $0.15, plus a network fee of $0.50, totaling $0.65. The protocol needs to pay an additional estimated $4 (accounting for the costs we initially considered negligible) in CCIP fee token for each unsuccessful raffle creation attempt.

Although the cost per transaction might appear insignificant, the cumulative effect of repeated failed attempts can become substantial. Moreover, if this exploit is used to repeatedly disrupt the core functionality of the protocol, leading to a denial of service (DoS), it becomes a serious issue that needs to be addressed.
## Impact

Denial Of Service + loss of funds
## Code Snippet

https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/81b28633d0f450e33a8b32976e17122418f5d47e/public-contracts/contracts/WinnablesTicketManager.sol#L278

https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/81b28633d0f450e33a8b32976e17122418f5d47e/public-contracts/contracts/WinnablesTicketManager.sol#L434

https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/81b28633d0f450e33a8b32976e17122418f5d47e/public-contracts/contracts/WinnablesTicketManager.sol#L264
## Tool used

Manual Review

## Recommendation

Consider adding a check in `_checkShouldCancel` so that only admin can cancel a raffle with `PRIZE_LOCKED` status

```diff
function cancelRaffle(address prizeManager, uint64 chainSelector, uint256 raffleId) external {
-       _checkShouldCancel(raffleId);
+       _checkShouldCancel(raffleId, msg.sender);

        _raffles[raffleId].status = RaffleStatus.CANCELED;
        _sendCCIPMessage(
            prizeManager,
            chainSelector,
            abi.encodePacked(uint8(CCIPMessageType.RAFFLE_CANCELED), raffleId)
        );
        IWinnablesTicket(TICKETS_CONTRACT).refreshMetadata(raffleId);
    }
```

```diff
 function _checkShouldCancel(uint256 raffleId, address caller) internal view {
        Raffle storage raffle = _raffles[raffleId];
-       if (raffle.status == RaffleStatus.PRIZE_LOCKED) return;
+       if (raffle.status == RaffleStatus.PRIZE_LOCKED && _hasRole(caller, 0)) return;
        if (raffle.status != RaffleStatus.IDLE) revert InvalidRaffle();
        if (raffle.endsAt > block.timestamp) revert RaffleIsStillOpen();
        uint256 supply = IWinnablesTicket(TICKETS_CONTRACT).supplyOf(raffleId);
        if (supply > raffle.minTicketsThreshold) revert TargetTicketsReached();
    }
```