# M-01: Manager cut is sent to `ContestManager` instead of the owner/admin

## Summary

`Pot.closePot()` transfers the manager cut to `msg.sender`. In the normal protocol flow, `Pot.closePot()` is called by `ContestManager`, so `msg.sender` inside `Pot` is the `ContestManager` contract, not the owner/admin EOA. Since `ContestManager` has no withdrawal function, the manager cut becomes stuck.

## Vulnerability Details

The intended assumption is that the manager receives 10% of the remaining pool when a pot is closed.

The close flow goes through `ContestManager`:

```solidity
function closeContest(address contest) public onlyOwner {
    _closeContest(contest);
}

function _closeContest(address contest) internal {
    Pot pot = Pot(contest);
    pot.closePot();
}
```

Inside `Pot.closePot()`, the manager cut is sent to `msg.sender`:

```solidity
uint256 managerCut = remainingRewards / managerCutPercent;
i_token.transfer(msg.sender, managerCut);
```

When `Pot.closePot()` is called through `ContestManager.closeContest()`, `msg.sender` is `ContestManager`. It is not the owner/admin who called `ContestManager.closeContest()`.

The result is:

```text
Pot -> ContestManager
```

instead of:

```text
Pot -> owner/admin
```

`ContestManager` does not implement a withdrawal or sweep function, so the ERC20 tokens sent there cannot be recovered through the protocol.

## Impact

The manager does not receive their intended 10% cut. The manager cut is transferred to `ContestManager` and becomes stuck.

This happens during the normal close path, so the likelihood is high. Every pot closed through `ContestManager.closeContest()` sends the cut to the wrong recipient.

## Proof of Concept

Example:

```text
remainingRewards = 1000
managerCutPercent = 10
managerCut = 100
```

The owner/admin calls:

```solidity
contestManager.closeContest(contest);
```

Call context:

```text
Owner EOA -> ContestManager.closeContest()
ContestManager -> Pot.closePot()
```

Inside `Pot.closePot()`:

```text
msg.sender == address(ContestManager)
```

Therefore:

```solidity
i_token.transfer(msg.sender, managerCut);
```

sends the 100-token manager cut to `ContestManager`, not to the owner/admin.

A Foundry-style test:

```solidity
function testManagerCutIsSentToContestManagerInsteadOfOwner() public {
    address alice = makeAddr("alice");

    address[] memory players = new address[](1);
    players[0] = alice;

    uint256[] memory rewards = new uint256[](1);
    rewards[0] = 100;

    uint256 totalRewards = 100;

    address contest = contestManager.createContest(
        players,
        rewards,
        IERC20(weth),
        totalRewards
    );

    ERC20Mock(weth).approve(address(contestManager), totalRewards);
    contestManager.fundContest(0);

    vm.warp(block.timestamp + 91 days);

    uint256 ownerBefore = ERC20Mock(weth).balanceOf(address(this));
    uint256 managerBefore = ERC20Mock(weth).balanceOf(address(contestManager));

    contestManager.closeContest(contest);

    assertEq(ERC20Mock(weth).balanceOf(address(contestManager)) - managerBefore, 10);
    assertEq(ERC20Mock(weth).balanceOf(address(this)) - ownerBefore, 0);
}
```

## Recommended Mitigation

Store the intended manager/owner recipient explicitly and transfer the manager cut to that address, not to `msg.sender` inside `Pot`.

One approach is to pass the manager address when deploying the pot:

```diff
contract Pot is Ownable(msg.sender) {
+   address private immutable i_manager;

-   constructor(address[] memory players, uint256[] memory rewards, IERC20 token, uint256 totalRewards) {
+   constructor(address[] memory players, uint256[] memory rewards, IERC20 token, uint256 totalRewards, address manager) {
        i_players = players;
        i_rewards = rewards;
        i_token = token;
        i_totalRewards = totalRewards;
+       i_manager = manager;
    }
}
```

Then pay `i_manager` during close:

```diff
- i_token.transfer(msg.sender, managerCut);
+ i_token.transfer(i_manager, managerCut);
```

In `ContestManager.createContest()`, pass `owner()` as the manager recipient:

```diff
- Pot pot = new Pot(players, rewards, token, totalRewards);
+ Pot pot = new Pot(players, rewards, token, totalRewards, owner());
```

Alternatively, make `ContestManager` capable of forwarding or withdrawing received manager cuts, but explicitly storing the intended recipient is clearer.

## Learning Notes

This finding reinforced that `msg.sender` is local to the immediate call. Contracts should not assume `msg.sender` is the original EOA after a cross-contract call.

Relevant concepts: call context, `msg.sender`, cross-contract accounting, stuck funds.

## Analysis Disclosure

Post-results learning.
