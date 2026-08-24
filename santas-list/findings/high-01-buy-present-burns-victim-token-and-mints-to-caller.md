Title: `buyPresent()` burns tokens from `presentReceiver` but mints the NFT to the caller instead of the receiver
Impact: High
Likelihood: High
Scope: SantasList.sol, SantaToken.sol

# Root + Impact

## Description

The intended behavior is that a user trades their own `SantaToken` balance for an additional NFT present. The README states that `buyPresent` trades `2e18` SantaTokens for an NFT, and the function comment says the caller must approve the `SantasList` contract to spend their SantaTokens.

The implementation instead accepts an arbitrary `presentReceiver` address, burns tokens from that address, and then mints the NFT to `msg.sender`. This charges the receiver rather than the caller, and sends the purchased NFT to the caller rather than the receiver. Since `SantaToken.burn()` can only be called by `SantasList`, the ERC20 allowance system is bypassed entirely once the call enters `SantasList.buyPresent()`.

```solidity
function buyPresent(address presentReceiver) external {
    // @> Burns from an arbitrary address supplied by the caller
    i_santaToken.burn(presentReceiver);
    // @> Mints the NFT to msg.sender, not to presentReceiver
    _mintAndIncrement();
}

function burn(address from) external {
    if (msg.sender != i_santasList) {
        revert SantaToken__NotSantasList();
    }
    // @> No allowance or ownership check for `from`
    _burn(from, 1e18);
}
```

## Risk

**Likelihood**:

* This occurs whenever any user has a SantaToken balance and another address calls `buyPresent(user)`.

* EXTRA_NICE users receive SantaTokens through normal protocol flow, so eligible victim balances are expected to exist.

**Impact**:

* Attackers can spend another user's SantaToken without approval.

* Attackers receive the purchased NFT while the victim loses their SantaToken.

* The protocol charges only `1e18` SantaToken even though the documented purchase cost is `2e18`.

## Proof of Concept

Add this test to `test/unit/SantasListTest.t.sol`:

```solidity
function testAttackerCanBurnVictimSantaTokenAndReceiveNft() public {
    address victim = makeAddr("victim");
    address attacker = makeAddr("attacker");

    vm.prank(santa);
    santasList.checkList(victim, SantasList.Status.EXTRA_NICE);

    vm.prank(santa);
    santasList.checkTwice(victim, SantasList.Status.EXTRA_NICE);

    vm.warp(santasList.CHRISTMAS_2023_BLOCK_TIME());

    vm.prank(victim);
    santasList.collectPresent();

    assertEq(santaToken.balanceOf(victim), 1e18);
    assertEq(santasList.balanceOf(attacker), 0);

    vm.prank(attacker);
    santasList.buyPresent(victim);

    assertEq(santaToken.balanceOf(victim), 0);
    assertEq(santasList.balanceOf(attacker), 1);
}
```

The victim receives a SantaToken by being `EXTRA_NICE`. The attacker then calls `buyPresent(victim)`, which burns the victim's token and mints the NFT to the attacker.

## Recommended Mitigation

Burn tokens from the caller, require the intended purchase cost, and mint the NFT to the caller or to an explicit receiver depending on the intended UX. If ERC20 approval is intended, use `transferFrom` or a burn function that checks allowance.

```diff
-function buyPresent(address presentReceiver) external {
-    i_santaToken.burn(presentReceiver);
-    _mintAndIncrement();
-}
+function buyPresent() external {
+    i_santaToken.burn(msg.sender, PURCHASED_PRESENT_COST);
+    _mintAndIncrement();
+}
```

```diff
-function burn(address from) external {
+function burn(address from, uint256 amount) external {
     if (msg.sender != i_santasList) {
         revert SantaToken__NotSantasList();
     }
-    _burn(from, 1e18);
+    _burn(from, amount);
 }
```
