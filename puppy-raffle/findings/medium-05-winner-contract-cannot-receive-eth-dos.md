Title: Winner contracts that cannot receive ETH can DoS winner selection
Impact: Medium
Likelihood: Medium
Scope: `src/PuppyRaffle.sol`

# Root + Impact

## Description

After the raffle duration passes, `selectWinner` should select a winner, pay the prize pool, mint the puppy NFT, and reset the raffle.

Players are accepted as arbitrary addresses, including smart contracts. If the selected winner is a contract that cannot receive plain ETH because it has no payable `receive` or `fallback` function, the low-level call used to send the prize pool fails. The whole `selectWinner` transaction reverts, so the raffle cannot be settled in that transaction.

```solidity
function selectWinner() external {
    ...
    address winner = players[winnerIndex];
    ...
    delete players;
    raffleStartTime = block.timestamp;
    previousWinner = winner;

    // @> Reverts when winner is a contract that rejects ETH.
    (bool success,) = winner.call{value: prizePool}("");
    require(success, "PuppyRaffle: Failed to send prize pool to winner");

    _safeMint(winner, tokenId);
}
```

## Risk

**Likelihood**:

* `enterRaffle` does not prevent contract addresses from being players.

* Many smart contracts do not implement payable `receive` or `fallback` functions.

**Impact**:

* Winner selection can revert when the selected winner cannot receive ETH.

* The raffle round can remain unsettled until a later call produces a different winner or the issue is otherwise worked around.

## Proof of Concept

```solidity
contract CannotReceiveEth {
    // No receive() and no payable fallback().
}

function testWinnerContractThatCannotReceiveEthRevertsSelectWinner() public {
    CannotReceiveEth badWinner = new CannotReceiveEth();

    address[] memory players = new address[](4);
    players[0] = address(1);
    players[1] = address(2);
    players[2] = address(3);
    players[3] = address(badWinner);
    puppyRaffle.enterRaffle{value: entranceFee * 4}(players);

    vm.warp(block.timestamp + duration + 1);
    vm.roll(block.number + 1);

    address biasedCaller;
    for (uint160 i = 100; i < 1000; i++) {
        address candidate = address(i);
        uint256 winnerIndex = uint256(
            keccak256(abi.encodePacked(candidate, block.timestamp, block.difficulty))
        ) % players.length;
        if (players[winnerIndex] == address(badWinner)) {
            biasedCaller = candidate;
            break;
        }
    }

    vm.prank(biasedCaller);
    vm.expectRevert("PuppyRaffle: Failed to send prize pool to winner");
    puppyRaffle.selectWinner();
}
```

When the selected winner cannot receive ETH, the payout call fails and the whole winner-selection flow reverts.

## Recommended Mitigation

Use a pull-payment pattern. Record the prize owed to the winner and let the winner withdraw it separately. This prevents one rejecting receiver from blocking winner selection and NFT minting.

```diff
+ mapping(address => uint256) public pendingPrizes;

 function selectWinner() external {
     ...
-    (bool success,) = winner.call{value: prizePool}("");
-    require(success, "PuppyRaffle: Failed to send prize pool to winner");
+    pendingPrizes[winner] += prizePool;
     _safeMint(winner, tokenId);
 }

+function withdrawPrize() external {
+    uint256 amount = pendingPrizes[msg.sender];
+    require(amount > 0, "PuppyRaffle: No prize");
+    pendingPrizes[msg.sender] = 0;
+    (bool success,) = msg.sender.call{value: amount}("");
+    require(success, "PuppyRaffle: Failed to withdraw prize");
+}
```
