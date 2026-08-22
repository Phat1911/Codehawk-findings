# M-02: `matchRewards()` can let users receive later dates without contributing funds

## Summary

`matchRewards()` uses one aggregate `userBalances[user]` value for all of a user's previous likes. When one match occurs, the function transfers and clears the user's entire balance, including funds paid for other unmatched likes. This allows a user to later match with another person while contributing no funds to that second match.

## Vulnerability Details

The intended protocol behavior is that like payments create meaningful on-chain commitments. When two users mutually like each other, their previous like payments are pooled for that match, minus a 10% fee.

The reward function does not account per pair or per liked user:

```solidity
function matchRewards(address from, address to) internal {
    uint256 matchUserOne = userBalances[from];
    uint256 matchUserTwo = userBalances[to];

    // Clears the entire balances of both matched users
    userBalances[from] = 0;
    userBalances[to] = 0;

    uint256 totalRewards = matchUserOne + matchUserTwo;
    uint256 matchingFees = (totalRewards * FIXEDFEE) / 100;
    uint256 rewards = totalRewards - matchingFees;
    totalFees += matchingFees;

    MultiSigWallet multiSigWallet = new MultiSigWallet(from, to);
    (bool success,) = payable(address(multiSigWallet)).call{value: rewards}("");
    require(success, "Transfer failed");
}
```

Because the balance is global per user, one match can consume funds that were paid while liking other users. After the balance is zeroed, a later mutual match can be created without that user contributing any remaining funds.

This is separate from the missing `userBalances[msg.sender] += msg.value` bug. Even after adding that missing credit, the accounting design still clears all funds across all pending likes.

## Impact

A user can get a later date funded mostly or entirely by the other participant. The protocol no longer ensures that every match is backed by both users' paid commitments.

Funds can also be misallocated to the wrong shared multisig because payments made while liking one user can be swept into a different match.

## Proof of Concept

Assume the intended accounting fix has been applied so each like credits `userBalances[msg.sender] += msg.value`.

```solidity
function testUserCanGetSecondDateWithoutContributingFunds() public {
    // Alice likes Bob with 1 ETH.
    registry.likeUser{value: 1 ether}(bob);

    // Alice also likes Charlie with 1 ETH before either user likes back.
    registry.likeUser{value: 1 ether}(charlie);

    // Alice's aggregate balance is now 2 ETH.
    assertEq(registry.userBalances(alice), 2 ether);

    // Bob likes Alice. The Alice/Bob match consumes Alice's full 2 ETH,
    // including Alice's payment intended for Charlie.
    registry.likeUser{value: 1 ether}(alice);
    assertEq(registry.userBalances(alice), 0);

    // Charlie later likes Alice. Alice contributes 0 ETH to this match
    // because her balance was already cleared by the Bob match.
    registry.likeUser{value: 1 ether}(alice);
}
```

Concrete flow:

```text
Alice pays 1 ETH to like Bob
Alice pays 1 ETH to like Charlie
Bob likes Alice
matchRewards(Bob, Alice) sends Bob's payment plus all 2 ETH from Alice to the Bob/Alice multisig
Charlie likes Alice
Alice's balance is already zero, so Alice gets a Charlie/Alice match without contributing funds to that match
```

## Recommended Mitigation

Track deposited value per liker and liked user, then consume only the funds for the matched pair.

```diff
-mapping(address => uint256) public userBalances;
+mapping(address => mapping(address => uint256)) public likeBalances;
```

```diff
 function likeUser(address liked) external payable {
     require(msg.value >= 1 ether, "Must send at least 1 ETH");
     require(!likes[msg.sender][liked], "Already liked");
     require(msg.sender != liked, "Cannot like yourself");
     require(profileNFT.profileToToken(msg.sender) != 0, "Must have a profile NFT");
     require(profileNFT.profileToToken(liked) != 0, "Liked user must have a profile NFT");

+    likeBalances[msg.sender][liked] += msg.value;
     likes[msg.sender][liked] = true;
```

```diff
-function matchRewards(address from, address to) internal {
-    uint256 matchUserOne = userBalances[from];
-    uint256 matchUserTwo = userBalances[to];
-    userBalances[from] = 0;
-    userBalances[to] = 0;
+function matchRewards(address from, address to) internal {
+    uint256 matchUserOne = likeBalances[from][to];
+    uint256 matchUserTwo = likeBalances[to][from];
+    likeBalances[from][to] = 0;
+    likeBalances[to][from] = 0;
```

## Learning Notes

Per-user balances are not enough when funds are tied to specific counterparties. The storage key should match the economic promise being made by the protocol.

Relevant concepts: accounting granularity, fund misallocation, protocol invariant failure.

## Analysis Disclosure

Missed finding.
