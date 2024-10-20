# Mystery Box - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Anyone can change owner](#H-01)
    - ### [H-02. Reentrancy attack in `MysteryBox::claimAllRewards` allows attacker to drain contract balance](#H-02)
    - ### [H-03. Reentrancy attack in `MysteryBox::claimSingleReward` allows attacker to drain contract balance](#H-03)
- ## Medium Risk Findings
    - ### [M-01. `randomValue` in `MysteryBox::openBox` is not random](#M-01)
    - ### [M-02. The `rewardPool` isn't used, so adding rewards doesn't impact the rewards players can receive](#M-02)
    - ### [M-03. A malicious user can expand the rewards array to an arbitrary size, significantly increasing storage costs](#M-03)
    - ### [M-04. Low contract balance can block players from claming their rewards](#M-04)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #25

### Dates: Sep 26th, 2024 - Oct 3rd, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-09-mystery-box)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 3
- Medium: 4
- Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Anyone can change owner            



## Relevant GitHub Links

<https://github.com/Cyfrin/2024-09-mystery-box/blob/main/src/MysteryBox.sol#L111>

# Summary

Anyone can call the `MysteryBox::changeOwner` function and change the contract’s owner. After that, the new owner (attacker) gains access to all `onlyOwner` functions and can steal funds by calling `MysteryBox::withdrawFunds`.

## Impact

An attacker can exploit this by calling `MysteryBox::changeOwner`, setting themselves as the owner, and then calling `MysteryBox::withdrawFunds` to steal funds.

## Recommendations

Add a check in the `MysteryBox::changeOwner` function to verify that the `msg.sender` is the current owner:

```diff
function changeOwner(address _newOwner) public {
+ 	require(msg.sender == owner, "Only owner can change owner");
    owner = _newOwner;
}
```

## <a id='H-02'></a>H-02. Reentrancy attack in `MysteryBox::claimAllRewards` allows attacker to drain contract balance            



## Relevant GitHub Links

<https://github.com/Cyfrin/2024-09-mystery-box/blob/main/src/MysteryBox.sol#L79-L90>

# Summary

The `MysteryBox::claimAllRewards` function does not follow the CEIFREI-PI principles, and, as a result, enables participants to drain the contract balance.

In the current implementation of `MysteryBox::claimAllRewards`, rewards are transferred first, and only then is `rewardsOwned[msg.sender]` reset.

A malicious player who has participated in the prediction could have a `fallback` or `receive` function that triggers the `MysteryBox::claimAllRewards` function again, allowing them to claim multiple refunds. They could repeat this process until the contract’s balance is completely drained.

## Impact

All the funds paid by players could be stolen by a malicious actor.

## Recommendations

To resolve this issue, the `MysteryBox::claimAllRewards` function should update the `rewardsOwned` array before making any external calls.

```diff
function claimAllRewards() public {
    uint256 totalValue = 0;
    for (uint256 i = 0; i < rewardsOwned[msg.sender].length; i++) {
        totalValue += rewardsOwned[msg.sender][i].value;
    }
    require(totalValue > 0, "No rewards to claim");
+   delete rewardsOwned[msg.sender];

    (bool success,) = payable(msg.sender).call{value: totalValue}("");
    require(success, "Transfer failed");

-   delete rewardsOwned[msg.sender];
}
```

## <a id='H-03'></a>H-03. Reentrancy attack in `MysteryBox::claimSingleReward` allows attacker to drain contract balance            



## Relevant GitHub Links

<https://github.com/Cyfrin/2024-09-mystery-box/blob/main/src/MysteryBox.sol#L92-L101>

# Summary

The `MysteryBox::claimSingleReward` function does not follow the CEIFREI-PI principles, and, as a result, enables participants to drain the contract balance.

In the current implementation of `MysteryBox::claimSingleReward`, rewards are transferred first, and only then is `rewardsOwned[msg.sender][_index]` reset.

A malicious player who has participated in the prediction could have a `fallback` or `receive` function that triggers the `MysteryBox::claimSingleReward` function again, allowing them to claim multiple refunds. They could repeat this process until the contract’s balance is completely drained.

## Impact

All the funds paid by players could be stolen by a malicious actor.

## Recommendations

To resolve this issue, the `MysteryBox::claimSingleReward` function should update the `rewardsOwned` array before making any external calls:

```diff
function claimSingleReward(uint256 _index) public {
    require(_index <= rewardsOwned[msg.sender].length, "Invalid index");
    uint256 value = rewardsOwned[msg.sender][_index].value;
    require(value > 0, "No reward to claim");
+   delete rewardsOwned[msg.sender][_index];

    (bool success,) = payable(msg.sender).call{value: value}("");
    require(success, "Transfer failed");

-   delete rewardsOwned[msg.sender][_index];
}
```

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. `randomValue` in `MysteryBox::openBox` is not random            



## Relevant GitHub Links

<https://github.com/Cyfrin/2024-09-mystery-box/blob/main/src/MysteryBox.sol#L47>

## Summary

The `MysteryBox::openBox` function uses `block.timestamp` and `msg.sender` to calculate a random value. These values can be manipulated by an attacker to increase their rewards.

## Impact

An attacker can manipulate the inputs used for random value calculation, allowing them to receive higher rewards.

## Recommendations

Integrate Chainlink VRF for secure random value generation. Implement two functions: one to request the random value (by calling `requestRandomWords` on the VRF Coordinator contract), and another to store the random value, triggered by the oracle (`fulfillRandomWords`).

## <a id='M-02'></a>M-02. The `rewardPool` isn't used, so adding rewards doesn't impact the rewards players can receive            



## Summary

When the owner adds new rewards using the `MysteryBox::addReward` function, the rewards available to players don’t change because the rewards are hardcoded in the `MysteryBox::openBox` function. Additionally, the hardcoded rewards in the `MysteryBox::openBox` function don’t match the rewards initialized in the `rewardPool` within the constructor (the same reward names have different values).

## Impact

Players will receive the same rewards even if the owner adds new ones.

If a player calls the `MysteryBox::getRewardPool` function to check available rewards, they’ll see incorrect values.

## Recommendations

Update the `MysteryBox::openBox` function to pull rewards from the `rewardPool`.

## <a id='M-03'></a>M-03. A malicious user can expand the rewards array to an arbitrary size, significantly increasing storage costs            



## Summary

After oppening one box, malicious user can expand rewards array to arbitrary size by transfering reward back and forth. This happens because `delete rewardsOwned[msg.sender][_index];` doesn't delete element in rewards array, nor just reset it and transfered reward is pushed to the rewards array.&#x20;

## Impact

A malicious user can expand the rewards array to an arbitrary size, significantly increasing storage costs

## Recommendations

Implement a delete function that shifts elements and uses `pop` to remove the reward from the array.

## <a id='M-04'></a>M-04. Low contract balance can block players from claming their rewards            



## Summary

If the owner only sends 0.1 ether to the contract during the constructor and the first player wins a reward larger than 0.1 ether, the player won’t be able to claim their reward due to insufficient contract balance. Additionally, the owner can't easily add more funds to the contract because there’s no payable function for funding it directly (the owner could repeatedly call `buyBox`, but that’s not the intended purpose of this function).

## Impact

Players may be unable to claim their rewards.

## Recommendations

Prevent users from buying a box if there’s a risk that the contract won’t have enough funds to pay the rewards and add funding function. The new condition should be added to the `MysteryBox::buyBox` function: `require((numberOfBoxesInCirculation + 1) * maxReward <= address(this).balance + boxPrice)`





