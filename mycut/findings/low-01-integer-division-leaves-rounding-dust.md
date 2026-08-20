# L-01: Integer division leaves rounding dust stuck in `Pot`

## Summary

`Pot.closePot()` uses integer division to calculate the manager cut and claimant payouts. Any remainder from these divisions is not transferred or accounted for, so small amounts of reward tokens can remain stuck in the pot.

## Vulnerability Details

The manager cut is calculated with integer division:

```solidity
uint256 managerCut = remainingRewards / managerCutPercent;
```

The claimant payout is also calculated with integer division:

```solidity
uint256 claimantCut = (remainingRewards - managerCut) / i_players.length;
```

Solidity integer division rounds down. The rounded-away amount is not sent to the manager, claimants, or any treasury address.

## Impact

Small amounts of tokens can remain permanently stuck in `Pot`. The impact is usually low because the stuck value is often dust, but it can become more noticeable for low-decimal tokens or across many contests.

## Proof of Concept

Example:

```text
remainingRewards = 101
managerCutPercent = 10
players.length = 4
```

Manager cut:

```text
101 / 10 = 10
```

Remaining amount:

```text
101 - 10 = 91
```

Claimant cut:

```text
91 / 4 = 22
```

Total distributed:

```text
manager cut: 10
claimants: 22 * 4 = 88
total: 98
```

Dust stuck:

```text
101 - 98 = 3
```

The remaining 3 tokens are never swept.

## Recommended Mitigation

Track the amount distributed and send any final remainder to a defined recipient.

```diff
uint256 managerCut = remainingRewards / managerCutPercent;
i_token.transfer(manager, managerCut);

uint256 rewardsForClaimants = remainingRewards - managerCut;
uint256 claimantCut = rewardsForClaimants / claimants.length;
+ uint256 distributed = managerCut;

for (uint256 i = 0; i < claimants.length; i++) {
    _transferReward(claimants[i], claimantCut);
+   distributed += claimantCut;
}

+ uint256 dust = remainingRewards - distributed;
+ if (dust > 0) {
+     i_token.transfer(manager, dust);
+ }
```

The protocol should explicitly define who receives rounding remainders.

## Learning Notes

This finding is mostly an accounting-quality issue. It is weaker than findings that lock material funds or break core distribution logic, but it is still useful as a reminder that integer division always truncates.

Relevant concepts: integer division, rounding dust, token accounting.

## Analysis Disclosure

Independent review.
