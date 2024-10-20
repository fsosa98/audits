# President Elector - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. The first president can be changed at any time](#H-01)
- ## Medium Risk Findings
    - ### [M-01. An attacker can reuse a voter's signature and call `RankedChoice::rankCandidatesBySig` in different elections](#M-01)
    - ### [M-02. Incorrect type in TYPEHASH](#M-02)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #24

### Dates: Sep 12th, 2024 - Sep 19th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-09-president-elector)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1
- Medium: 2
- Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. The first president can be changed at any time            



## Relevant GitHub Links

<https://github.com/Cyfrin/2024-09-president-elector/blob/main/src/RankedChoice.sol#L61-L66>



# Summary

In the `RankedChoice::selectPresident` function, there is a condition that checks if 4 years have passed since the president was elected: `block.timestamp - s_previousVoteEndTimeStamp <= i_presidentalDuration`. For the first president, `s_previousVoteEndTimeStamp` is set to 0, and since `block.timestamp > i_presidentalDuration`, the `selectPresident` function won't revert, even though 4 years haven't passed yet, allowing a new president to be selected.

## Impact

Someone can replace the first president before their 4-year term ends.&#x20;

Furthermore, if a voter immediately after contract creation (before other voters submit their votes) calls `RankedChoice::rankCandidates` and then `RankedChoice::selectPresident`, they could select the president they prefer.

## Recommendations

In the constructor, update `s_previousVoteEndTimeStamp` to `block.timestamp`:

```diff
constructor(address[] memory voters) EIP712("RankedChoice", "1") {
    VOTERS = voters;
    i_presidentalDuration = 1460 days;
    s_currentPresident = msg.sender;
    s_voteNumber = 0;
+   s_previousVoteEndTimeStamp = block.timestamp;
}
```

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. An attacker can reuse a voter's signature and call `RankedChoice::rankCandidatesBySig` in different elections            



## Summary

An attacker can store the signature used in the `RankedChoice::rankCandidatesBySig` function and reuse it to call `RankedChoice::rankCandidatesBySig` in different elections, effectively overwriting the voter's vote.

## Impact

If the voter doesn’t cast a vote in a future election, the attacker can vote on their behalf using the candidates from previous elections (where the voter signed the voting message).

If the voter does vote, the attacker can modify their candidate list.

## Recommendations

The hashed message should also include `s_voteNumber` so the signature cannot be reused in different elections:

```diff
- bytes32 public constant TYPEHASH = keccak256("rankCandidates(uint256[])");
+ bytes32 public constant TYPEHASH = keccak256("rankCandidates(address[], uint256)");

function rankCandidatesBySig(address[] memory orderedCandidates, bytes memory signature) external {
-  bytes32 structHash = keccak256(abi.encode(TYPEHASH, orderedCandidates));
+  bytes32 structHash = keccak256(abi.encode(TYPEHASH, orderedCandidates, s_voteNumber));
   bytes32 hash = _hashTypedDataV4(structHash);
   address signer = ECDSA.recover(hash, signature);
   _rankCandidates(orderedCandidates, signer);
}
```

## <a id='M-02'></a>M-02. Incorrect type in TYPEHASH            



## Relevant GitHub Links

<https://github.com/Cyfrin/2024-09-president-elector/blob/main/src/RankedChoice.sol#L23>

## Summary

The `TYPEHASH` variable is defined as the hash of the `RankedChoice::rankCandidates` function selector. However, in `TYPEHASH`, the parameter is incorrectly defined as `uint256[]`, while in the `RankedChoice::rankCandidates` function, the parameter is actually `address[]`.

## Impact

When looking at the `RankedChoice::rankCandidates` function, a voter might sign a hashed message with a different function selector (`keccak256("rankCandidates(address[])")`), which could block them from voting in the election.

## Recommendations

Update `TYPEHASH` to the correct value:

```diff
- bytes32 public constant TYPEHASH = keccak256("rankCandidates(uint256[])");
+ bytes32 public constant TYPEHASH = keccak256("rankCandidates(address[])");
```





