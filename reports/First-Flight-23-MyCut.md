# MyCut - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Incorrect calculation of claimantCut in the Pot::closePot function](#H-01)
    - ### [H-02. The manager's cut gets stuck in the `ContestManager` contract](#H-02)
- ## Medium Risk Findings
    - ### [M-01. Players don’t have 90 days to claim rewards if the manager doesn’t fund the contest immediately after creation](#M-01)
- ## Low Risk Findings
    - ### [L-01. Tokens can get stuck in the Pot contract](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #23

### Dates: Aug 29th, 2024 - Sep 5th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-08-MyCut)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 2
- Medium: 1
- Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. Incorrect calculation of claimantCut in the Pot::closePot function            



## Relevant GitHub Links

<https://github.com/Cyfrin/2024-08-MyCut/blob/main/src/Pot.sol#L57>



## Summary

The project description states that the remaining rewards should be distributed equally among those who claimed them in time. However, in the `Pot::closePot` function, the formula for `claimantCut` is `(remainingRewards - managerCut) / i_players.length`. If not all players claim their rewards (i.e., `claimants.length < i_players.length`), the claimants will receive a smaller cut, and tokens will remain in the Pot contract after the pot is closed.

## Impact

If not all players claim their rewards, they will receive a smaller cut when the pot is closed.

## Recommendation

To fix this, the denominator in the formula should be changed to `claimants.length`:

```diff
- uint256 claimantCut = (remainingRewards - managerCut) / i_players.length;
+ uint256 claimantCut = (remainingRewards - managerCut) / claimants.length;
```

## <a id='H-02'></a>H-02. The manager's cut gets stuck in the `ContestManager` contract            



## Summary

When the manager creates a contest using the `ContestManager::createContest` function, the owner of the `Pot` contract is set to the `ContestManager` contract. The `Pot::closePot` function can only be called by the owner, which is the `ContestManager` contract. The manager's cut is transferred to `msg.sender`, which in this case is the `ContestManager` contract, and there is no function in the `ContestManager` contract to retrieve the tokens.

## Impact

The manager's cut gets stuck in the `ContestManager` contract after closing the pot.

## Recommendation

Add a function to the `ContestManager` contract to retrieve the tokens.

```solidity
function getTokens(IERC20 token) external onlyOwner {
	token.transfer(msg.sender, token.balanceOf(address(this)));
}
```

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Players don’t have 90 days to claim rewards if the manager doesn’t fund the contest immediately after creation            



## Summary

Players can claim rewards only after the manager funds the contest. However, the 90-day limit is calculated from the time of contest creation, not from when the contest is funded.

## Impact

Players have less than 90 days to claim their rewards if the manager does not fund the contest immediately.

## Recommendation

Funding the contest should be done in the `Pot` contract constructor, or there should be a fund function that updates `i_deployedAt`.


# Low Risk Findings

## <a id='L-01'></a>L-01. Tokens can get stuck in the Pot contract            



## Summary

If `remainingRewards - managerCut < i_players.length` and `remainingRewards < managerCutPercent`, tokens will get stuck in the contract because both `managerCut` and `claimantCut` will be 0. The maximum balance that can get stuck in the `Pot` contract is 9.

## Impact

Tokens become stuck in the `Pot` contract.

## Recommendations

If `remainingRewards - managerCut < i_players.length`, transfer all `remainingRewards` to the manager, because claimants can't receive less than 1 token. After closing the Pot, update the `remainingRewards` to 0.



