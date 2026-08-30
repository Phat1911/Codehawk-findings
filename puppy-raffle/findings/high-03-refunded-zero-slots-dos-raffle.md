Title: Refunded players are replaced with `address(0)`, permanently DoSing raffle entry and winner selection
Impact: High
Likelihood: High
Scope: `src/PuppyRaffle.sol`

# Root + Impact

## Description

Refunded players should no longer participate in the raffle, should not block future entries, and should no longer be counted as funds available for the prize pool.

`refund` replaces a refunded player's slot with `address(0)` but does not remove the slot from the array. Once two players refund, the `players` array contains duplicate `address(0)` entries. Because `enterRaffle` checks duplicates across the entire `players` array, every future entry reverts with `PuppyRaffle: Duplicate player`.

The same stale slots also break winner selection. `selectWinner` calculates `totalAmountCollected` as `players.length * entranceFee`, treating refunded slots as paid active entries. When enough players refund, the contract calculates a prize pool larger than the ETH it actually holds, causing winner payout to revert. If a zero slot is selected as the winner, `_safeMint(address(0), tokenId)` also reverts.

```solidity
function enterRaffle(address[] memory newPlayers) public payable {
    ...
    // @> After two refunds, duplicate address(0) slots make this revert.
    for (uint256 i = 0; i < players.length - 1; i++) {
        for (uint256 j = i + 1; j < players.length; j++) {
            require(players[i] != players[j], "PuppyRaffle: Duplicate player");
        }
    }
}

function refund(uint256 playerIndex) public {
    ...
    payable(msg.sender).sendValue(entranceFee);

    // @> The array length is unchanged, leaving a blank slot.
    players[playerIndex] = address(0);
    emit RaffleRefunded(playerAddress);
}

function selectWinner() external {
    ...
    // @> Refunded slots are still counted as paid entries.
    uint256 totalAmountCollected = players.length * entranceFee;
    uint256 prizePool = (totalAmountCollected * 80) / 100;
    ...
    (bool success,) = winner.call{value: prizePool}("");
    require(success, "PuppyRaffle: Failed to send prize pool to winner");
}
```

## Risk

**Likelihood**:

* Refunds intentionally leave blank spots in the `players` array.

* After two refunds, `players` contains duplicate `address(0)` entries.

* `enterRaffle` always checks duplicates across the entire array, including refunded slots.

* `selectWinner` always uses `players.length` instead of counting active nonzero players or actual contract balance.

**Impact**:

* Future users cannot enter the raffle because duplicate `address(0)` slots make `enterRaffle` revert.

* Winner selection can revert because the calculated prize pool exceeds the contract balance.

* Winner selection can also revert when `address(0)` is selected as the winner and `_safeMint` rejects minting to the zero address.

* A participant can make the raffle round impossible to progress normally after refunds occur.

## Proof of Concept

```solidity
function testTwoRefundsPermanentlyDosFutureEntries() public {
    address[] memory players = new address[](4);
    players[0] = address(1);
    players[1] = address(2);
    players[2] = address(3);
    players[3] = address(4);
    puppyRaffle.enterRaffle{value: entranceFee * 4}(players);

    for (uint256 i = 0; i < 2; i++) {
        vm.prank(players[i]);
        puppyRaffle.refund(i);
    }

    address[] memory newPlayers = new address[](1);
    newPlayers[0] = address(5);

    vm.expectRevert("PuppyRaffle: Duplicate player");
    puppyRaffle.enterRaffle{value: entranceFee}(newPlayers);
}

function testRefundedPlayersCanDosSelectWinner() public {
    address[] memory players = new address[](4);
    players[0] = address(1);
    players[1] = address(2);
    players[2] = address(3);
    players[3] = address(4);
    puppyRaffle.enterRaffle{value: entranceFee * 4}(players);

    for (uint256 i = 0; i < 3; i++) {
        vm.prank(players[i]);
        puppyRaffle.refund(i);
    }

    assertEq(address(puppyRaffle).balance, entranceFee);

    vm.warp(block.timestamp + duration + 1);
    vm.roll(block.number + 1);

    vm.expectRevert("PuppyRaffle: Failed to send prize pool to winner");
    puppyRaffle.selectWinner();
}
```

After two refunds, `players[0]` and `players[1]` are both `address(0)`, so the duplicate check prevents all future entries. After three refunds, `players.length` is still `4`, so `selectWinner` tries to pay `3.2 ETH` even though the contract only has `1 ETH`.

## Recommended Mitigation

Remove refunded players from the active participant set instead of leaving duplicate zero entries. Track active paid entries separately or calculate rewards from actual available ETH. Also ensure `address(0)` cannot win.

```diff
+ uint256 public activePlayerCount;
+ mapping(address => bool) public isActivePlayer;

 function enterRaffle(address[] memory newPlayers) public payable {
     require(msg.value == entranceFee * newPlayers.length, "PuppyRaffle: Must send enough to enter raffle");
     for (uint256 i = 0; i < newPlayers.length; i++) {
+        require(!isActivePlayer[newPlayers[i]], "PuppyRaffle: Duplicate player");
+        isActivePlayer[newPlayers[i]] = true;
         players.push(newPlayers[i]);
+        activePlayerCount++;
     }
     ...
 }

 function refund(uint256 playerIndex) public {
     ...
     players[playerIndex] = address(0);
+    isActivePlayer[msg.sender] = false;
+    activePlayerCount--;
     payable(msg.sender).sendValue(entranceFee);
 }

 function selectWinner() external {
     ...
-    uint256 totalAmountCollected = players.length * entranceFee;
+    uint256 totalAmountCollected = activePlayerCount * entranceFee;
     ...
 }
```
