# L-02: `encodeTokenId()` and `decodeTokenId()` are inconsistent for large inputs

## Summary

`encodeTokenId()` accepts `uint256 collectionId` and `uint256 itemId`, but `decodeTokenId()` only recovers a 128-bit `itemId`. For large `itemId` values, decoding an encoded token ID does not return the original inputs.

## Vulnerability Details

The intended token layout is:

```text
upper 128 bits = collectionId
lower 128 bits = itemId
```

The implementation is:

```solidity
function encodeTokenId(uint256 collectionId, uint256 itemId) public pure returns (uint256) {
    return (collectionId << COLLECTION_ID_SHIFT) + itemId;
}

function decodeTokenId(uint256 tokenId) public pure returns (uint256 collectionId, uint256 itemId) {
    collectionId = tokenId >> COLLECTION_ID_SHIFT;
    itemId = uint256(uint128(tokenId));
}
```

When `itemId > type(uint128).max`, the value spills into the collection portion. The public function signature allows this even though the encoding format cannot preserve it.

## Impact

External callers or integrations using the helper with large `uint256` inputs can receive incorrect decoded collection and item values. In normal protocol minting, reaching such large item IDs is practically impossible, so this is low severity.

## Proof of Concept

```solidity
function testEncodeDecodeMismatchForLargeItemId() public view {
    uint256 collectionId = 100;
    uint256 itemId = uint256(type(uint128).max) + 1;

    uint256 tokenId = festivalPass.encodeTokenId(collectionId, itemId);
    (uint256 decodedCollectionId, uint256 decodedItemId) = festivalPass.decodeTokenId(tokenId);

    assertEq(decodedCollectionId, 101);
    assertEq(decodedItemId, 0);
}
```

## Recommended Mitigation

Validate the 128-bit bounds or expose the bound in the function signature.

```diff
 function encodeTokenId(uint256 collectionId, uint256 itemId) public pure returns (uint256) {
-    return (collectionId << COLLECTION_ID_SHIFT) + itemId;
+    require(collectionId <= type(uint128).max, "Collection ID too large");
+    require(itemId <= type(uint128).max, "Item ID too large");
+    return (collectionId << COLLECTION_ID_SHIFT) | itemId;
 }
```

## Learning Notes

Helper APIs should make their bit-width assumptions explicit. Otherwise, valid Solidity inputs can produce surprising outputs.

Relevant concepts: bit packing, truncation, helper invariant.

## Analysis Disclosure

Independent review.
