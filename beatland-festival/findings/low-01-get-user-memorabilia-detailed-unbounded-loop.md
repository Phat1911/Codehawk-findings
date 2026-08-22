# L-01: Unbounded nested loops can make `getUserMemorabiliaDetailed()` unusable

## Summary

`getUserMemorabiliaDetailed()` scans every collection and every minted item twice. As the protocol grows, the view call can become too expensive for RPC providers or on-chain integrations. This is downgraded because it affects a helper view function rather than the core state-changing flows.

## Vulnerability Details

The function counts matching memorabilia with a nested scan:

```solidity
for (uint256 cId = 1; cId < nextCollectionId; cId++) {
    for (uint256 iId = 1; iId < collections[cId].currentItemId; iId++) {
        uint256 tokenId = encodeTokenId(cId, iId);
        if (balanceOf(user, tokenId) > 0) {
            count++;
        }
    }
}
```

It then repeats the same nested scan to populate arrays:

```solidity
for (uint256 cId = 1; cId < nextCollectionId; cId++) {
    for (uint256 iId = 1; iId < collections[cId].currentItemId; iId++) {
        uint256 tokenId = encodeTokenId(cId, iId);
        if (balanceOf(user, tokenId) > 0) {
            tokenIds[index] = tokenId;
            collectionIds[index] = cId;
            itemIds[index] = iId;
            index++;
        }
    }
}
```

The cost grows with global collections and global minted items, not only with the queried user's holdings.

## Impact

Frontends and users may become unable to retrieve memorabilia details through this helper as the number of collections and items grows. Core actions like buying passes, attending performances, and redeeming memorabilia remain available.

## Proof of Concept

```solidity
function testGetUserMemorabiliaDetailedCostGrowsWithGlobalState() public {
    address alice = makeAddr("alice");

    vm.startPrank(organizer);
    for (uint256 i = 0; i < 50; i++) {
        festivalPass.createMemorabiliaCollection("Collection", "ipfs://collection", 1e18, 100, true);
    }
    vm.stopPrank();

    vm.prank(address(festivalPass));
    beatToken.mint(alice, 10_000e18);

    vm.startPrank(alice);
    for (uint256 collectionId = 100; collectionId < 150; collectionId++) {
        for (uint256 i = 0; i < 10; i++) {
            festivalPass.redeemMemorabilia(collectionId);
        }
    }
    vm.stopPrank();

    festivalPass.getUserMemorabiliaDetailed(alice);
}
```

## Recommended Mitigation

Track user-owned memorabilia directly when minting, or expose paginated queries.

```diff
+mapping(address user => uint256[] tokenIds) private userMemorabilia;

 _mint(msg.sender, tokenId, 1, "");
+userMemorabilia[msg.sender].push(tokenId);
```

```diff
+function getUserMemorabilia(address user) external view returns (uint256[] memory) {
+    return userMemorabilia[user];
+}
```

## Learning Notes

View-only loops can still hurt usability, but they are usually lower severity when no critical state transition depends on them.

Relevant concepts: unbounded loops, view function scalability, pagination.

## Analysis Disclosure

Independent review, downgraded.
