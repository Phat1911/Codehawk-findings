# M-01: `configurePass()` resets sold supply while existing passes remain minted

## Summary

`configurePass()` resets `passSupply[passId]` to zero every time it is called. Existing ERC1155 passes are not burned. If a pass is reconfigured after sales, the contract forgets already minted passes and can sell more than the configured max supply.

## Vulnerability Details

The organizer can configure pass price and maximum supply:

```solidity
function configurePass(
    uint256 passId,
    uint256 price,
    uint256 maxSupply
) external onlyOrganizer {
    require(passId == GENERAL_PASS || passId == VIP_PASS || passId == BACKSTAGE_PASS, "Invalid pass ID");
    require(price > 0, "Price must be greater than 0");
    require(maxSupply > 0, "Max supply must be greater than 0");

    passPrice[passId] = price;
    passMaxSupply[passId] = maxSupply;
    passSupply[passId] = 0;
}
```

`buyPass()` enforces the max supply with that reset counter:

```solidity
require(passSupply[collectionId] < passMaxSupply[collectionId], "Max supply reached");
_mint(msg.sender, collectionId, 1, "");
++passSupply[collectionId];
```

Reconfiguration does not burn existing passes or read actual ERC1155 supply, so live supply accounting can become lower than the number of minted passes.

## Impact

The protocol can oversell limited pass tiers. Scarcity guarantees for VIP or backstage passes can be broken through normal reconfiguration.

## Proof of Concept

```solidity
function testConfigurePassResetAllowsOverselling() public {
    address alice = makeAddr("alice");
    address bob = makeAddr("bob");
    vm.deal(alice, 1 ether);
    vm.deal(bob, 1 ether);

    vm.prank(organizer);
    festivalPass.configurePass(3, BACKSTAGE_PRICE, 1);

    vm.prank(alice);
    festivalPass.buyPass{value: BACKSTAGE_PRICE}(3);

    vm.prank(organizer);
    festivalPass.configurePass(3, BACKSTAGE_PRICE, 1);

    vm.prank(bob);
    festivalPass.buyPass{value: BACKSTAGE_PRICE}(3);

    assertEq(festivalPass.balanceOf(alice, 3), 1);
    assertEq(festivalPass.balanceOf(bob, 3), 1);
    assertEq(festivalPass.passMaxSupply(3), 1);
}
```

## Recommended Mitigation

Do not reset `passSupply` during reconfiguration. Also prevent setting `maxSupply` below already minted supply.

```diff
 require(price > 0, "Price must be greater than 0");
 require(maxSupply > 0, "Max supply must be greater than 0");
+require(maxSupply >= passSupply[passId], "Max supply below minted supply");
 
 passPrice[passId] = price;
 passMaxSupply[passId] = maxSupply;
-passSupply[passId] = 0;
```

## Learning Notes

Admin configuration should update parameters without corrupting live accounting. Existing minted supply must remain part of max-supply enforcement.

Relevant concepts: supply accounting, admin reconfiguration, ERC1155 scarcity.

## Analysis Disclosure

Independent review.
