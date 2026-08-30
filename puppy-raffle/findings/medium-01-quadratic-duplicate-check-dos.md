Title: Quadratic duplicate checks can DoS raffle entry
Impact: Medium
Likelihood: High
Scope: `src/PuppyRaffle.sol`

# Root + Impact

## Description

`enterRaffle` should let users enter one or more participants while rejecting duplicate addresses.

The duplicate check compares every player against every other player in the entire `players` array after appending the new players. This is an `O(n^2)` operation. As the raffle grows, entering becomes increasingly expensive and can eventually exceed the block gas limit, preventing new users from entering.

```solidity
function enterRaffle(address[] memory newPlayers) public payable {
    require(msg.value == entranceFee * newPlayers.length, "PuppyRaffle: Must send enough to enter raffle");
    for (uint256 i = 0; i < newPlayers.length; i++) {
        players.push(newPlayers[i]);
    }

    // @> Nested loop over all players makes entry cost grow quadratically.
    for (uint256 i = 0; i < players.length - 1; i++) {
        for (uint256 j = i + 1; j < players.length; j++) {
            require(players[i] != players[j], "PuppyRaffle: Duplicate player");
        }
    }
    emit RaffleEnter(newPlayers);
}
```

## Risk

**Likelihood**:

* Every raffle entry executes the nested duplicate check.

* The gas cost increases rapidly as more players join the same raffle round.

**Impact**:

* Users can be prevented from entering once the array is large enough.

* The protocol loses availability for one of its main functions.

## Proof of Concept

The following test shows the gas cost rising as the number of existing players increases:

```solidity
function testEnterRaffleGasGrowsQuadratically() public {
    uint256 smallStartGas;
    uint256 smallGasUsed;
    uint256 largeStartGas;
    uint256 largeGasUsed;

    for (uint160 i = 1; i <= 20; i++) {
        address[] memory player = new address[](1);
        player[0] = address(i);
        puppyRaffle.enterRaffle{value: entranceFee}(player);
    }

    address[] memory smallEntry = new address[](1);
    smallEntry[0] = address(10_000);
    smallStartGas = gasleft();
    puppyRaffle.enterRaffle{value: entranceFee}(smallEntry);
    smallGasUsed = smallStartGas - gasleft();

    for (uint160 i = 21; i <= 200; i++) {
        address[] memory player = new address[](1);
        player[0] = address(i);
        puppyRaffle.enterRaffle{value: entranceFee}(player);
    }

    address[] memory largeEntry = new address[](1);
    largeEntry[0] = address(20_000);
    largeStartGas = gasleft();
    puppyRaffle.enterRaffle{value: entranceFee}(largeEntry);
    largeGasUsed = largeStartGas - gasleft();

    assertGt(largeGasUsed, smallGasUsed);
}
```

With enough players, the same operation becomes too expensive for normal users or impossible within a block.

## Recommended Mitigation

Track active players with a mapping and check only the new entrants instead of comparing every pair in the full array.

```diff
+ mapping(address => bool) public isActivePlayer;

 function enterRaffle(address[] memory newPlayers) public payable {
     require(msg.value == entranceFee * newPlayers.length, "PuppyRaffle: Must send enough to enter raffle");
     for (uint256 i = 0; i < newPlayers.length; i++) {
+        require(!isActivePlayer[newPlayers[i]], "PuppyRaffle: Duplicate player");
+        isActivePlayer[newPlayers[i]] = true;
         players.push(newPlayers[i]);
     }
-
-    for (uint256 i = 0; i < players.length - 1; i++) {
-        for (uint256 j = i + 1; j < players.length; j++) {
-            require(players[i] != players[j], "PuppyRaffle: Duplicate player");
-        }
-    }
     emit RaffleEnter(newPlayers);
 }
```
