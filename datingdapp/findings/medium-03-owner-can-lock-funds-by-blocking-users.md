# M-03: App owner can lock users' deposited funds by blocking profiles

## Summary

Users' paid likes have no withdrawal or refund path. The app owner can call `blockProfile()` on a user who has pending paid likes, removing the profile NFT needed to complete future matches. Those pending funds can no longer be matched and cannot be withdrawn by the user.

## Vulnerability Details

`LikeRegistry.likeUser()` requires both the liker and liked address to have active profile NFTs:

```solidity
require(profileNFT.profileToToken(msg.sender) != 0, "Must have a profile NFT");
require(profileNFT.profileToToken(liked) != 0, "Liked user must have a profile NFT");
```

The only code path that sends user funds into a multisig is a future mutual match:

```solidity
if (likes[liked][msg.sender]) {
    matches[msg.sender].push(liked);
    matches[liked].push(msg.sender);
    emit Matched(msg.sender, liked);
    matchRewards(liked, msg.sender);
}
```

There is no function for users to cancel a like, withdraw unmatched funds, or receive a refund when a profile is blocked. The owner can burn a user's profile with:

```solidity
function blockProfile(address blockAddress) external onlyOwner {
    uint256 tokenId = profileToToken[blockAddress];
    require(tokenId != 0, "No profile found");

    _burn(tokenId);
    delete profileToToken[blockAddress];
    delete _profiles[tokenId];

    emit ProfileBurned(blockAddress, tokenId);
}
```

Once a relevant profile is removed, the profile-gated matching path is broken while the previously deposited like funds remain in the registry.

## Impact

The app owner can cause users' pending like payments to become locked by blocking a participant before the mutual match is completed. The affected user has no self-service recovery path.

This is especially harmful because the protocol requires users to pay ETH upfront to express interest.

## Proof of Concept

The example below assumes the intended balance credit exists. In the submitted code, the situation is worse because paid ETH is not credited at all and is already stuck.

```solidity
function testOwnerCanLockPendingLikeFundsByBlockingUser() public {
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

    nft.blockProfile(bob);

    vm.prank(bob);
    vm.expectRevert("Must have a profile NFT");
    registry.likeUser{value: 1 ether}(alice);

    // Alice has no cancel/refund/withdraw function to recover the pending like payment.
}
```

## Recommended Mitigation

Provide a refund path for pending likes and handle refunds during profile blocking.

One approach is to track pending balances per pair and allow cancellation before a match:

```diff
+function cancelLike(address liked) external {
+    require(likes[msg.sender][liked], "Like not found");
+    require(!likes[liked][msg.sender], "Already matched");
+
+    likes[msg.sender][liked] = false;
+    uint256 amount = likeBalances[msg.sender][liked];
+    likeBalances[msg.sender][liked] = 0;
+
+    (bool success,) = payable(msg.sender).call{value: amount}("");
+    require(success, "Refund failed");
+}
```

When blocking users, also refund their pending balances or mark them claimable through a pull-payment withdrawal.

## Learning Notes

Admin moderation can affect fund liveness. A profile-gated protocol that accepts upfront payments should define what happens to pending funds when a profile is burned, blocked, or otherwise removed.

Relevant concepts: admin risk, stuck funds, refund paths, liveness.

## Analysis Disclosure

Missed finding.
