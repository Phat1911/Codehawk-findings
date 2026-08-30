Title: Solidity `<0.8.0` arithmetic and `uint64` fee accounting can overflow and permanently lock fees
Impact: Medium
Likelihood: Medium
Scope: `src/PuppyRaffle.sol`

# Root + Impact

## Description

After each raffle, the protocol should safely account for 20% of the collected funds as fees and allow those fees to be withdrawn to `feeAddress`.

The contract uses Solidity `0.7.6`, so arithmetic does not automatically revert on overflow or underflow. This means any addition, multiplication, or narrowing cast must be protected manually with SafeMath-style checks.

The concrete vulnerable path is fee accounting. `totalFees` is stored as `uint64`, but fees are denominated in wei. `uint64` can store only about `18.44 ether`. When the fee amount exceeds the `uint64` range, `uint64(fee)` truncates the value and records the wrong fee amount. The subsequent addition to `totalFees` can also wrap because it is Solidity `<0.8.0` arithmetic.

```solidity
// @> Too small for wei-denominated ETH accounting.
uint64 public totalFees = 0;

function selectWinner() external {
    ...
    uint256 fee = (totalAmountCollected * 20) / 100;

    // @> fee is cast from uint256 to uint64 and can truncate.
    // @> The addition can also overflow because this is Solidity 0.7.6.
    totalFees = totalFees + uint64(fee);
    ...
}

function withdrawFees() external {
    // @> This equality can never pass once totalFees is corrupted.
    require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");
    ...
}
```

## Risk

**Likelihood**:

* The contract is compiled with Solidity `0.7.6`, where arithmetic overflow and underflow do not automatically revert.

* Raffles collect ETH in wei, while `uint64` has a low maximum for ETH-denominated accounting.

* A raffle with more than about `92.23 ether` total collected creates a fee greater than `type(uint64).max`.

**Impact**:

* `totalFees` becomes smaller than the actual ETH fee balance.

* `withdrawFees` can permanently revert, locking protocol fees in the contract.

## Proof of Concept

With an entrance fee of `1 ether`, 93 players produce an `18.6 ether` fee, which exceeds `type(uint64).max`.

```solidity
function testTotalFeesOverflowLocksFees() public {
    address[] memory players = new address[](93);
    for (uint160 i = 0; i < 93; i++) {
        players[i] = address(i + 1);
    }

    puppyRaffle.enterRaffle{value: entranceFee * 93}(players);

    vm.warp(block.timestamp + duration + 1);
    vm.roll(block.number + 1);
    puppyRaffle.selectWinner();

    uint256 expectedFee = (entranceFee * 93 * 20) / 100;
    assertEq(address(puppyRaffle).balance, expectedFee);
    assertLt(puppyRaffle.totalFees(), expectedFee);

    vm.expectRevert("PuppyRaffle: There are currently players active!");
    puppyRaffle.withdrawFees();
}
```

The balance contains the real fee amount, but `totalFees` holds a truncated value, so the strict equality check fails.

## Recommended Mitigation

Use `uint256` for wei-denominated accounting and use SafeMath or checked arithmetic for Solidity `0.7.6`. Avoid narrowing casts for monetary values.

```diff
 pragma solidity ^0.7.6;
+import {SafeMath} from "@openzeppelin/contracts/math/SafeMath.sol";

 contract PuppyRaffle is ERC721, Ownable {
+    using SafeMath for uint256;
- uint64 public totalFees = 0;
+ uint256 public totalFees = 0;

 function selectWinner() external {
     ...
     uint256 fee = (totalAmountCollected * 20) / 100;
-    totalFees = totalFees + uint64(fee);
+    totalFees = totalFees.add(fee);
     ...
 }
```
