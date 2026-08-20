# H-01: `closePot()` can be permanently DoS'd with a large claimant list

## Summary

`Pot.closePot()` pays every claimant in a single loop. If enough users claim before the 90-day deadline, the close transaction can exceed the block gas limit and revert forever, preventing final settlement of the pot.

## Vulnerability Details

The intended flow is that users claim during the 90-day window, then the owner/admin closes the pot. During close, the manager receives a cut of the remaining pool and the rest is redistributed to users who claimed in time.

That final close step is implemented as a push-payment loop:

```solidity
uint256 claimantCut = (remainingRewards - managerCut) / i_players.length;
for (uint256 i = 0; i < claimants.length; i++) {
    _transferReward(claimants[i], claimantCut);
}
```

The `claimants` array grows every time an authorized player calls `claimCut()`:

```solidity
claimants.push(player);
```

There is no upper bound on the number of players/claimants. As the claimant list grows, `closePot()` becomes more expensive. With a large enough list, the function can no longer fit inside the block gas limit.

## Impact

The owner/admin may be unable to close the pot. This prevents the manager cut and post-deadline redistribution from being executed and can leave remaining reward tokens stuck in the pot.

This is high impact because the final settlement path can become permanently unavailable for a valid contest with many claimants.

## Proof of Concept

Consider a contest with a very large number of authorized players. Each player claims during the 90-day claim period, causing their address to be appended to `claimants`.

After 90 days, the owner calls `closeContest()`, which calls `Pot.closePot()`. `closePot()` must loop over the entire `claimants` array and perform an ERC20 transfer for every claimant.

For small claimant counts this succeeds:

```text
claimants.length = 10
closePot() loops 10 times
```

For large claimant counts this becomes impossible:

```text
claimants.length = thousands of users
closePot() attempts thousands of ERC20 transfers
transaction exceeds block gas limit
closePot() reverts
```

Because there is no batched close function, pull-payment mechanism, or progress-tracking index, every future call tries to process the same full claimant list again and reverts again.

## Recommended Mitigation

Avoid paying all claimants in a single unbounded loop. Use a pull-based model or a batched settlement model.

One approach is to finalize the pot during `closePot()` without transferring to every claimant immediately:

```diff
+ bool private closed;
+ uint256 private finalClaimantBonus;

function closePot() external onlyOwner {
    if (block.timestamp - i_deployedAt < 90 days) {
        revert Pot__StillOpenForClaim();
    }
+   closed = true;

    if (remainingRewards > 0) {
        uint256 managerCut = remainingRewards / managerCutPercent;
        i_token.transfer(manager, managerCut);

-       uint256 claimantCut = (remainingRewards - managerCut) / i_players.length;
-       for (uint256 i = 0; i < claimants.length; i++) {
-           _transferReward(claimants[i], claimantCut);
-       }
+       finalClaimantBonus = (remainingRewards - managerCut) / claimants.length;
    }
}
```

Then allow each eligible claimant to withdraw their bonus individually. This keeps each transaction bounded and avoids making pot closure depend on the size of the claimant array.

## Learning Notes

This finding shows why settlement functions should avoid unbounded push-payment loops. Even if every individual transfer is valid, batching too many transfers into one required transaction can break protocol liveness.

Relevant concepts: gas DoS, unbounded loops, push vs pull payments, settlement liveness.

## Analysis Disclosure

Post-results learning.
