# M-01: Blocked users can mint a new profile and bypass moderation

## Summary

`SoulboundProfileNFT.blockProfile()` burns a user's current profile NFT and deletes their `profileToToken` mapping entry. Since `mintProfile()` only checks that `profileToToken[msg.sender] == 0`, a blocked user can immediately mint a new profile and regain access to profile-gated app actions.

## Vulnerability Details

The owner-facing function is intended to let the app owner block users:

```solidity
/// @notice App owner can block users
function blockProfile(address blockAddress) external onlyOwner {
    uint256 tokenId = profileToToken[blockAddress];
    require(tokenId != 0, "No profile found");

    _burn(tokenId);
    delete profileToToken[blockAddress];
    delete _profiles[tokenId];

    emit ProfileBurned(blockAddress, tokenId);
}
```

The only eligibility check in `mintProfile()` is the deleted mapping entry:

```solidity
function mintProfile(string memory name, uint8 age, string memory profileImage) external {
    require(profileToToken[msg.sender] == 0, "Profile already exists");

    uint256 tokenId = ++_nextTokenId;
    _safeMint(msg.sender, tokenId);

    _profiles[tokenId] = Profile(name, age, profileImage);
    profileToToken[msg.sender] = tokenId;

    emit ProfileMinted(msg.sender, tokenId, name, age, profileImage);
}
```

There is no `blocked` mapping or other permanent moderation state. Burning the NFT removes the current profile but also restores the user's ability to mint.

## Impact

The app owner's blocking action does not persist. A blocked user can mint again and satisfy the profile checks in `LikeRegistry.likeUser()`:

```solidity
require(profileNFT.profileToToken(msg.sender) != 0, "Must have a profile NFT");
require(profileNFT.profileToToken(liked) != 0, "Liked user must have a profile NFT");
```

This weakens moderation and allows blocked users back into profile-gated protocol flows.

## Proof of Concept

```solidity
function testBlockedUserCanMintAgain() public {
    SoulboundProfileNFT nft = new SoulboundProfileNFT();
    address alice = address(0xA11CE);

    vm.prank(alice);
    nft.mintProfile("Alice", 25, "ipfs://alice");
    assertEq(nft.profileToToken(alice), 1);

    nft.blockProfile(alice);
    assertEq(nft.profileToToken(alice), 0);

    vm.prank(alice);
    nft.mintProfile("Alice again", 25, "ipfs://alice-again");

    assertEq(nft.profileToToken(alice), 2);
}
```

## Recommended Mitigation

Track blocked users independently from the currently active profile token.

```diff
 contract SoulboundProfileNFT is ERC721, Ownable {
+    mapping(address => bool) public blocked;

     function mintProfile(string memory name, uint8 age, string memory profileImage) external {
+        require(!blocked[msg.sender], "Profile blocked");
         require(profileToToken[msg.sender] == 0, "Profile already exists");
```

```diff
 function blockProfile(address blockAddress) external onlyOwner {
     uint256 tokenId = profileToToken[blockAddress];
     require(tokenId != 0, "No profile found");

+    blocked[blockAddress] = true;
     _burn(tokenId);
     delete profileToToken[blockAddress];
     delete _profiles[tokenId];
```

Add an owner-controlled unblock function only when temporary moderation is intended.

## Learning Notes

Deleting active user state is not the same as recording a ban. Moderation systems need durable state for the permission decision they intend to enforce.

Relevant concepts: access control, moderation state, authorization bypass.

## Analysis Disclosure

Independent review.
