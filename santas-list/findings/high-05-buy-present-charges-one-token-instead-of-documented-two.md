Title: `buyPresent()` charges `1e18` SantaToken despite the documented `2e18` purchase cost
Impact: High
Likelihood: High
Scope: SantasList.sol, SantaToken.sol

# Root + Impact

## Description

The intended behavior is that buying an additional present costs `2e18` SantaTokens. The README states that `buyPresent` trades `2e18` SantaToken for an NFT, and `SantasList` defines `PURCHASED_PRESENT_COST` as `2e18`.

The implementation never uses `PURCHASED_PRESENT_COST`. Instead, `buyPresent()` calls `SantaToken.burn()`, and `SantaToken.burn()` always burns a fixed `1e18` amount. As a result, presents are sold for half of the documented protocol price.

```solidity
uint256 public constant PURCHASED_PRESENT_COST = 2e18;

function buyPresent(address presentReceiver) external {
    // @> PURCHASED_PRESENT_COST is not passed or enforced
    i_santaToken.burn(presentReceiver);
    _mintAndIncrement();
}
```

```solidity
function burn(address from) external {
    if (msg.sender != i_santasList) {
        revert SantaToken__NotSantasList();
    }
    // @> Always burns 1e18, not the documented 2e18 cost
    _burn(from, 1e18);
}
```

## Risk

**Likelihood**:

* This occurs on every successful `buyPresent()` call.

* The purchase cost constant exists in `SantasList`, but no code path uses it.

**Impact**:

* Users can buy presents for half of the intended SantaToken cost.

* The SantaToken economy is weakened because each token buys more NFTs than intended.

## Proof of Concept

Add this test to `test/unit/SantasListTest.t.sol`:

```solidity
function testBuyPresentChargesOneTokenInsteadOfDocumentedTwo() public {
    vm.startPrank(santa);
    santasList.checkList(user, SantasList.Status.EXTRA_NICE);
    santasList.checkTwice(user, SantasList.Status.EXTRA_NICE);
    vm.stopPrank();

    vm.warp(santasList.CHRISTMAS_2023_BLOCK_TIME() + 1);

    vm.startPrank(user);
    santasList.collectPresent();

    assertEq(santasList.PURCHASED_PRESENT_COST(), 2e18);
    assertEq(santaToken.balanceOf(user), 1e18);

    santasList.buyPresent(user);

    assertEq(santasList.balanceOf(user), 2);
    assertEq(santaToken.balanceOf(user), 0);
    vm.stopPrank();
}
```

The user only has `1e18` SantaToken, which is less than `PURCHASED_PRESENT_COST`, but `buyPresent()` still succeeds.

## Recommended Mitigation

Make `SantaToken.burn()` accept an amount and pass `PURCHASED_PRESENT_COST` from `buyPresent()`.

```diff
-function buyPresent(address presentReceiver) external {
-    i_santaToken.burn(presentReceiver);
+function buyPresent(address presentReceiver) external {
+    i_santaToken.burn(msg.sender, PURCHASED_PRESENT_COST);
     _mintAndIncrement();
 }
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
