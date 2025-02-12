# [H-1] A user can pay less in fees by vouching initially with a smaller amount and then using the `EthosVouch::increaseVouch` function to add the remaining vouch value

## Summary

A vulnerability in the `EthosVouch` fee mechanism allows users to reduce fees when vouching for a subject. By splitting their vouching process into multiple smaller transactions, users can partially reclaim `vouchersPoolFee`, resulting in significantly lower total fees compared to a single large transaction. This exploit undermines the intended fee structure and results in financial losses for other previous vouchers.

## Root Cause

The `vouchersPoolFee` is redistributed to existing vouchers. By vouching with a smaller value initially, a user becomes an existing voucher and subsequently benefits from `vouchersPoolFee` in subsequent [EthosVouch::increaseVouch](https://github.com/sherlock-audit/2024-11-ethos-network-ii/blob/main/ethos/packages/contracts/contracts/EthosVouch.sol#L426) calls. The logic does not distinguish between fees for new vouches and subsequent increases, enabling fee circumvention.

## External pre-conditions

There is at least one other existing voucher to receive part of the vouchersPoolFee.

## Attack Path

1. A user initially vouches with a smaller value (e.g., 10 ETH instead of the intended 100 ETH).
2. The user becomes an existing voucher and receives part of the `vouchersPoolFee`.
3. The user repeatedly calls `increaseVouch` in smaller increments (e.g., 10 ETH per transaction) to reach the intended total vouch value.
4. In each `increaseVouch` call, the user reclaims part of the `vouchersPoolFee`, significantly reducing the total fees paid.

## Impact

Example: User A wants to vouch with 100 ETH for a subject S. User B has already vouched with 1 ETH for that subject. The fees are defined as follows:

- entryProtocolFeeBasisPoints = 100 (1%)
- entryDonationFeeBasisPoints = 200 (2%)
- entryVouchersPoolFeeBasisPoints = 300 (3%)

If User A simply calls the `EthosVouch::vouchByProfileId` function with a `msg.value` of 100 ETH, they will pay approximately 0.99 ETH to the protocol, 1.96 ETH to the subject S, and 2.91 ETH to User B (the only previous voucher). This means they will pay a total of around 5.86 ETH in fees, and their vouch balance will be 94.14 ETH.

However, if User A wants to pay fewer fees, they can call the `EthosVouch::vouchByProfileId` function with a `msg.value` of 10 ETH and then call the `EthosVouch::increaseVouch` function nine more times, each with 10 ETH. By doing this, User A becomes a previous voucher and receives part of the `vouchersPoolFee`. In the end, their vouch balance will be approximately 96.68 ETH, meaning they paid 2.54 ETH less in fees (5.86 ETH - 3.32 ETH) compared to the first case.

Note: On-chain fees are excluded from the calculations, but they are much lower than the protocol fees.

## Mitigation

Exclude the vouching user from receiving vouchersPoolFee during their own transactions