# H-01: Paid like ETH is never credited, causing zero rewards and stuck funds

## Summary

`LikeRegistry.likeUser()` requires users to send at least 1 ETH, but never credits `msg.value` to `userBalances`. When a mutual match happens, `matchRewards()` reads zero balances, sends zero ETH to the shared multisig, records zero fees, and leaves the paid ETH stuck in `LikeRegistry`.

## Vulnerability Details

The README describes the core flow as users paying 1 ETH to like a profile, then having all previous like payments minus a 10% fee pooled into a shared multisig wallet when the like is mutual.

However, the payment is only accepted by `likeUser()`:

```solidity
function likeUser(address liked) external payable {
    require(msg.value >= 1 ether, "Must send at least 1 ETH");
    require(!likes[msg.sender][liked], "Already liked");
    require(msg.sender != liked, "Cannot like yourself");
    require(profileNFT.profileToToken(msg.sender) != 0, "Must have a profile NFT");
    require(profileNFT.profileToToken(liked) != 0, "Liked user must have a profile NFT");

    // msg.value is never added to userBalances[msg.sender]
    likes[msg.sender][liked] = true;
    emit Liked(msg.sender, liked);
```

`matchRewards()` later relies entirely on `userBalances`:

```solidity
function matchRewards(address from, address to) internal {
    uint256 matchUserOne = userBalances[from];
    uint256 matchUserTwo = userBalances[to];
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

Because `userBalances` is never increased, `totalRewards`, `matchingFees`, and `rewards` are always zero.

## Impact

Matched users receive no ETH in their shared multisig despite paying to like profiles. The protocol also records no fees, so `withdrawFees()` cannot recover the paid ETH. The ETH paid into `LikeRegistry` has no withdrawal path and becomes stuck.

This breaks the main advertised mechanism of the protocol.

## Proof of Concept

```solidity
function testPaidLikeEthIsNeverCredited() public {
    SoulboundProfileNFT nft = new SoulboundProfileNFT();
    LikeRegistry registry = new LikeRegistry(address(nft));

    address alice = address(0xA11CE);
    address bob = address(0xB0B);
    vm.deal(alice, 1 ether);
    vm.deal(bob, 1 ether);

    vm.prank(alice);
    nft.mintProfile("Alice", 25, "ipfs://alice");

    vm.prank(bob);
    nft.mintProfile("Bob", 26, "ipfs://bob");

    vm.prank(alice);
    registry.likeUser{value: 1 ether}(bob);

    assertEq(registry.userBalances(alice), 0);

    vm.prank(bob);
    registry.likeUser{value: 1 ether}(alice);

    assertEq(registry.userBalances(alice), 0);
    assertEq(registry.userBalances(bob), 0);
    assertEq(address(registry).balance, 2 ether);

    vm.expectRevert("No fees to withdraw");
    registry.withdrawFees();
}
```

## Recommended Mitigation

Credit the incoming payment before evaluating a mutual match.

```diff
 function likeUser(address liked) external payable {
     require(msg.value >= 1 ether, "Must send at least 1 ETH");
     require(!likes[msg.sender][liked], "Already liked");
     require(msg.sender != liked, "Cannot like yourself");
     require(profileNFT.profileToToken(msg.sender) != 0, "Must have a profile NFT");
     require(profileNFT.profileToToken(liked) != 0, "Liked user must have a profile NFT");

+    userBalances[msg.sender] += msg.value;
     likes[msg.sender][liked] = true;
     emit Liked(msg.sender, liked);
```

## Learning Notes

This finding shows why value transfer and accounting updates should be reviewed together. Accepting ETH without immediately recording its ownership usually creates either stuck funds or incorrect later settlement.

Relevant concepts: ETH accounting, reward accounting, stuck funds, protocol invariant failure.

## Analysis Disclosure

Independent review.
