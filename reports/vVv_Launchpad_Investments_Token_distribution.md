# [H-1] Claiming tokens can be front-run by an attacker

## Summary

In the VVVVCTokenDistributor::claim function, tokens are transferred to msg.sender: https://github.com/sherlock-audit/2024-11-vvv-exchange-update-fsosa98/blob/main/vvv-platform-smart-contracts/contracts/vc/VVVVCTokenDistributor.sol#L131-L135. An attacker can monitor the blockchain for ClaimParams objects and call the claim function using their own address.

## Root Cause

In [VVVVCTokenDistributor::claim](https://github.com/sherlock-audit/2024-11-vvv-exchange-update/blob/main/vvv-platform-smart-contracts/contracts/vc/VVVVCTokenDistributor.sol#L133), project tokens are transferred to msg.sender. If a malicious address front-runs the function call, it becomes the msg.sender and claims the tokens.

## Internal pre-conditions

The attacker obtains a valid unclaimed ClaimParams object

## Attack Path

Obtain ClaimParams: The attacker monitors the blockchain or intercepts transactions to retrieve a valid unclaimed ClaimParams object.
Front-Run the Transaction: The attacker submits a claim transaction with their own address as msg.sender, racing ahead of the legitimate user’s transaction.
Claim Tokens: As the msg.sender in the front-run transaction, the attacker successfully claims the tokens intended for the legitimate user.

## Impact

An attacker can intercept and claim tokens meant for legitimate users, causing unauthorized transfers and potential financial losses for both the users and the project.

## PoC
The attacker calls the VVVVCTokenDistributor::claim function before the actual claimer:

```solidity
function testFrontrunAttack() public {
    address[] memory thisProjectTokenProxyWallets = new address[](1);
    uint256[] memory thisTokenAmountsToClaim = new uint256[](1);

    thisProjectTokenProxyWallets[0] = projectTokenProxyWallets[0];

    uint256 claimAmount = sampleTokenAmountsToClaim[0];
    thisTokenAmountsToClaim[0] = claimAmount;

    VVVVCTokenDistributor.ClaimParams memory claimParams =
        generateClaimParamsWithSignature(sampleKycAddress, thisProjectTokenProxyWallets, thisTokenAmountsToClaim);

    address attacker = vm.addr(54321);
    claimAsUser(attacker, claimParams);
    assertTrue(ProjectTokenInstance.balanceOf(attacker) == claimAmount);
}
```

```
[PASS] testFrontrunAttack() (gas: 121982)
```

## Mitigation

Modify the VVVVCTokenDistributor::claim function to transfer tokens directly to the kycAddress specified in the ClaimParams object instead of msg.sender. This ensures tokens are sent to the intended recipient and prevents front-running attacks.

```diff
function claim(ClaimParams memory _params) public {
    ...
    // transfer tokens from each wallet to the caller
    for (uint256 i = 0; i < _params.projectTokenProxyWallets.length; i++) {
        projectToken.safeTransferFrom(
            _params.projectTokenProxyWallets[i],
-           msg.sender,
+	    _params.kycAddress,
            _params.tokenAmountsToClaim[i]
        );
    }
    ...
}
```