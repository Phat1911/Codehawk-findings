# H-03: Memorabilia collections can never mint their full configured max supply

## Summary

Memorabilia collections start `currentItemId` at 1, but `redeemMemorabilia()` requires `currentItemId < maxSupply`. This makes the final configured item impossible to mint. A collection with `maxSupply = 1` cannot mint any item.

## Vulnerability Details

Collections initialize the next item ID to 1:

```solidity
collections[collectionId] = MemorabiliaCollection({
    name: name,
    baseUri: baseUri,
    priceInBeat: priceInBeat,
    maxSupply: maxSupply,
    currentItemId: 1,
    isActive: activateNow
});
```

But redemption rejects `currentItemId == maxSupply`:

```solidity
require(collection.currentItemId < collection.maxSupply, "Collection sold out");
```

For `maxSupply = 10`, the contract mints item IDs 1 through 9 and rejects item 10. For `maxSupply = 1`, it rejects the first redemption immediately.

## Impact

Collections mint fewer items than promised. One-of-one memorabilia collections are completely unusable, and every collection underdelivers its configured supply.

## Proof of Concept

```solidity
function testMaxSupplyOneCannotMintAnyMemorabilia() public {
    address alice = makeAddr("alice");

    vm.prank(organizer);
    uint256 collectionId = festivalPass.createMemorabiliaCollection(
        "One of One",
        "ipfs://one",
        1e18,
        1,
        true
    );

    vm.prank(address(festivalPass));
    beatToken.mint(alice, 1e18);

    vm.prank(alice);
    vm.expectRevert("Collection sold out");
    festivalPass.redeemMemorabilia(collectionId);
}
```

## Recommended Mitigation

Allow `currentItemId` to equal `maxSupply`, since item IDs start at 1.

```diff
-require(collection.currentItemId < collection.maxSupply, "Collection sold out");
+require(collection.currentItemId <= collection.maxSupply, "Collection sold out");
```

## Learning Notes

Supply checks must match counter initialization. A counter starting at 1 usually needs `<= maxSupply` when minting the final item.

Relevant concepts: off-by-one errors, supply caps, NFT editions.

## Analysis Disclosure

Independent review.
