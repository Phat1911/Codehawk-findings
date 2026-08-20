# M-02: Duplicate player addresses overwrite assigned rewards

## Summary

`Pot` stores player rewards in a mapping. If the same player address appears multiple times in the `players` array, the later entry overwrites the earlier one instead of accumulating rewards or reverting.

## Vulnerability Details

The constructor receives parallel arrays of players and rewards:

```solidity
constructor(address[] memory players, uint256[] memory rewards, IERC20 token, uint256 totalRewards) {
    i_players = players;
    i_rewards = rewards;
    i_token = token;
    i_totalRewards = totalRewards;
    remainingRewards = totalRewards;
    i_deployedAt = block.timestamp;

    for (uint256 i = 0; i < i_players.length; i++) {
        playersToRewards[i_players[i]] = i_rewards[i];
    }
}
```

The issue is the assignment:

```solidity
playersToRewards[i_players[i]] = i_rewards[i];
```

Mappings can only store one value per key. If a player appears more than once, the latest value replaces the previous value.

Example:

```text
players = [Alice, Alice]
rewards = [100, 200]
```

The loop does:

```text
playersToRewards[Alice] = 100
playersToRewards[Alice] = 200
```

Alice can claim only `200`, not `300`, and the overwritten `100` remains accounted for in `totalRewards`/`remainingRewards` but is not claimable by Alice.

## Impact

Players can receive less than their intended reward allocation when duplicate entries exist. The overwritten rewards remain in the pot and distort later close accounting.

This can happen from bad off-chain input, duplicated contest data, or repeated eligible entries for the same address. Because the protocol does not validate uniqueness, the contract silently accepts the invalid state.

## Proof of Concept

Setup:

```text
players = [Alice, Alice]
rewards = [100, 200]
totalRewards = 300
```

Expected behavior could be either:

```text
Alice can claim 300
```

or:

```text
Contest creation reverts because duplicate players are invalid
```

Actual behavior:

```text
Alice can claim only 200
```

Foundry-style test:

```solidity
function testDuplicatePlayerAddressOverwritesReward() public {
    address alice = makeAddr("alice");

    address[] memory players = new address[](2);
    players[0] = alice;
    players[1] = alice;

    uint256[] memory rewards = new uint256[](2);
    rewards[0] = 100;
    rewards[1] = 200;

    uint256 totalRewards = 300;

    address contest = contestManager.createContest(
        players,
        rewards,
        IERC20(weth),
        totalRewards
    );

    ERC20Mock(weth).approve(address(contestManager), totalRewards);
    contestManager.fundContest(0);

    vm.prank(alice);
    Pot(contest).claimCut();

    assertEq(ERC20Mock(weth).balanceOf(alice), 200);
    assertEq(Pot(contest).getRemainingRewards(), 100);
}
```

## Recommended Mitigation

Reject duplicate player addresses during contest creation or accumulate duplicate rewards deliberately.

The safer approach is to require unique players:

```diff
for (uint256 i = 0; i < players.length; i++) {
+   if (players[i] == address(0)) revert ContestManager__InvalidPlayer();
+   for (uint256 j = i + 1; j < players.length; j++) {
+       if (players[i] == players[j]) revert ContestManager__DuplicatePlayer();
+   }
}
```

If duplicate entries are expected to represent multiple reward grants, accumulate instead:

```diff
- playersToRewards[i_players[i]] = i_rewards[i];
+ playersToRewards[i_players[i]] += i_rewards[i];
```

The protocol should choose one behavior explicitly instead of silently overwriting.

## Learning Notes

This finding shows why array-to-mapping initialization needs uniqueness checks. A mapping assignment can hide duplicate data errors that would be visible in an array.

Relevant concepts: mappings, duplicate input validation, reward accounting.

## Analysis Disclosure

Post-results learning.
