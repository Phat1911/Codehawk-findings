# M-03: Remaining rewards are divided by all players instead of successful claimants

## Summary

`Pot.closePot()` distributes the post-manager-cut remainder only to addresses in `claimants`, but calculates each payout using `i_players.length`. When only some players claimed in time, each timely claimant is underpaid and the undistributed tokens remain in the pot.

## Vulnerability Details

The README states that after 90 days, the manager takes a cut of the remaining pool and the remainder is distributed equally to those who claimed in time.

The implementation loops over `claimants`:

```solidity
for (uint256 i = 0; i < claimants.length; i++) {
    _transferReward(claimants[i], claimantCut);
}
```

But it calculates `claimantCut` using all original players:

```solidity
uint256 claimantCut = (remainingRewards - managerCut) / i_players.length;
```

The recipient set and denominator do not match. The function pays only claimants, but divides by all players.

## Impact

Timely claimants receive less than the amount promised by the protocol whenever not all players claimed before close. The leftover tokens are not redistributed and remain stuck in the pot.

## Proof of Concept

Assume:

```text
Total players: 4
Reward per player: 100
Total rewards: 400
```

Only Alice claims during the 90-day claim period.

After Alice claims:

```text
Alice receives: 100
remainingRewards = 300
claimants.length = 1
i_players.length = 4
```

After 90 days, the owner closes the pot:

```text
managerCut = 300 / 10 = 30
remaining for timely claimants = 270
```

Expected:

```text
Alice receives 270 because she is the only timely claimant
```

Actual:

```text
claimantCut = 270 / 4 = 67
Alice receives 67
203 tokens remain stuck
```

## Recommended Mitigation

Use `claimants.length` as the denominator because only claimants receive the remaining pool.

```diff
- uint256 claimantCut = (remainingRewards - managerCut) / i_players.length;
+ uint256 claimantCut = (remainingRewards - managerCut) / claimants.length;
```

Also handle the case where no users claimed in time:

```diff
+ if (claimants.length == 0) {
+     i_token.transfer(manager, remainingRewards - managerCut);
+     return;
+ }
```

The exact recipient for the no-claimant remainder should be defined by the protocol rules.

## Learning Notes

This finding highlights a common accounting mistake: the denominator must represent the same group that is being paid.

Relevant concepts: reward distribution, accounting invariants, stuck funds.

## Analysis Disclosure

Independent review.
