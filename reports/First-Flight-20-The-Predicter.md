# First Flight #20: The Predicter - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Player with only one correct paid prediction is not eligible for a reward](#H-01)
    - ### [H-02. Reentrancy attack in `ThePredicter::cancelRegistration` allows attacker to drain contract balance](#H-02)
    - ### [H-03. Everyone can make predictions – Players who didn't pay the entrance fee and aren't approved by the organizer can still make predictions](#H-03)
- ## Medium Risk Findings
    - ### [M-01. Players can't submit predictions until 19:00:00 UTC on the day of the match](#M-01)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #20

### Dates: Jul 18th, 2024 - Jul 25th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-07-the-predicter)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 3
- Medium: 1
- Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Player with only one correct paid prediction is not eligible for a reward            



## Relevant GitHub Links

<https://github.com/Cyfrin/2024-07-the-predicter/blob/main/src/ScoreBoard.sol#L97>

## Summary

One condition in `ScoreBoard::isEligibleForReward` function is that player need to make more then one prediction to be eligable for reward. In project description (`README.md`) there is description that player with one correct paid prediction should be eligable for reward: `who paid at least one prediction fee`.

One condition in the `ScoreBoard::isEligibleForReward` function is that a player needs to make more than one prediction to be eligible for a reward. In the project description (`README.md`), it states that a player with one correct paid prediction should be eligible for a reward: `prize fund, which after the end of the tournament is distributed among all Players who paid at least one prediction fee`.

## Impact

A player with one correct paid prediction can't withdraw their reward because they are not eligible.

## Recommendations

To fix this, we should change the condition in `ScoreBoard::isEligibleForReward` to include players with `predictionsCount` greater than or equal to 1.

```diff
function isEligibleForReward(address player) public view returns (bool) {
    return results[NUM_MATCHES - 1] != Result.Pending &&
-   playersPredictions[player].predictionsCount > 1; 
+   playersPredictions[player].predictionsCount >= 1; 
}
```

## <a id='H-02'></a>H-02. Reentrancy attack in `ThePredicter::cancelRegistration` allows attacker to drain contract balance            



## Relevant GitHub Links

<https://github.com/Cyfrin/2024-07-the-predicter/blob/main/src/ThePredicter.sol#L64>

## Summary

The `ThePredicter::cancelRegistration` function does not follow CEIFREI-PI principles and, as a result, enables participants to drain the contract balance.

In the `ThePredicter::cancelRegistration` function, we first make an external call to the `msg.sender` address, and only after making that external call, we update the `playersStatus` array.

A player who has entered the prediction could have a `fallback`/`receive` function that calls the `ThePredicter::cancelRegistration` function again and claim another refund. They could continue to cycle this until the contract balance is drained.

## Impact

All fees paid by players could be stolen by a malicious participant.

## Recommendations

To fix this, we should have the `ThePredicter::cancelRegistration` function update the `playersStatus` array before making the external call.

```diff
function cancelRegistration() public {
    if (playersStatus[msg.sender] == Status.Pending) {
+       playersStatus[msg.sender] = Status.Canceled;
        (bool success, ) = msg.sender.call{value: entranceFee}("");
        require(success, "Failed to withdraw");
-       playersStatus[msg.sender] = Status.Canceled;
        return;
    }
    revert ThePredicter__NotEligibleForWithdraw();
}
```

## <a id='H-03'></a>H-03. Everyone can make predictions – Players who didn't pay the entrance fee and aren't approved by the organizer can still make predictions            



## Summary

In `ThePredicter::makePrediction`, there is no restriction on any address making a prediction. This means that players who didn't pay the entrance fee and aren't approved by the organizer can make predictions.

These players can also withdraw their rewards (if they are eligible), and the withdraw function doesn't include their rewards in `totalShares`.

## Impact

* Users don't need to pay the entrance fee to make predictions and get rewards.
* Users don't need to be approved by the organizer to make predictions and get rewards.
* If the `predictionFee` is less than the `entranceFee`, users who made predictions without paying the entranceFee can block approved users from getting their rewards (because they are not included in `totalShares`).

## Recommendations

Add a condition to the function to check if the player is approved by the organizer.

```diff
function makePrediction(
    uint256 matchNumber,
    ScoreBoard.Result prediction
) public payable {
+   if (playersStatus[player] != Status.Approved) {
+       revert("Player not approved by organizer");
+	}
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

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Players can't submit predictions until 19:00:00 UTC on the day of the match            



## Relevant GitHub Links

<https://github.com/Cyfrin/2024-07-the-predicter/blob/main/src/ScoreBoard.sol#L66>

<https://github.com/Cyfrin/2024-07-the-predicter/blob/main/src/ThePredicter.sol#L93>

## Summary

In the `ScoreBoard::setPrediction` and `ThePredicter::makePrediction` functions, there is a condition that should check if a player can make a prediction due to restrictions that allow players to make predictions until 19:00:00 UTC on the day of the match. The condition in these functions is wrong for all matches.&#x20;

For example, for the first game, players should be able to submit predictions until 2024-08-15 19:00:00 UTC, but they cansubmit predictions only until 2024-08-15 01:00:00 UTC.

## Impact

Players can't submit predictions until 19:00:00 UTC on the day of the match.

## Recommendations

Replace the incorrect condition in the functions with the correct one. For example in `ScoreBoard::setPrediction` function:

```diff
function setPrediction(
    address player,
    uint256 matchNumber,
    Result result
) public {
-   if (block.timestamp <= START_TIME + matchNumber * 68400 - 68400)
+   if (block.timestamp <= START_TIME - 3600 + matchNumber * 86400)
       playersPredictions[player].predictions[matchNumber] = result;
    playersPredictions[player].predictionsCount = 0;
    for (uint256 i = 0; i < NUM_MATCHES; ++i) {
        if (
            playersPredictions[player].predictions[i] != Result.Pending &&
            playersPredictions[player].isPaid[i]
        ) ++playersPredictions[player].predictionsCount;
    }
}
```





