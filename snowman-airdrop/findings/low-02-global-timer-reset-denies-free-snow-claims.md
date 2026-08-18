# L-02: Global timer reset in `Snow::buySnow` denies free Snow claims for all users

## Summary

`Snow.buySnow()` updates the same global timer used by `earnSnow()`. As a result, any buyer can reset the free-earn cooldown for every user. Because `buySnow(0)` is allowed, this can be done at no cost.

## Vulnerability Details

The intended assumption is that users can earn free Snow once enough time has passed.

`earnSnow()` checks one shared timer:

```solidity
if (s_earnTimer != 0 && block.timestamp < (s_earnTimer + 1 weeks)) {
    revert S__Timer();
}
```

However, `buySnow()` also updates this timer:

```solidity
s_earnTimer = block.timestamp;
```

The problem occurs in `src/Snow.sol` inside `buySnow()` and `earnSnow()`. Since `s_earnTimer` is global, one user's buy action affects every user's ability to earn free Snow.

The issue is easier to trigger because `buySnow(0)` succeeds with no ETH or WETH payment. With `amount == 0`, the ETH branch condition becomes `0 == 0`, so the function mints zero tokens but still resets `s_earnTimer`.

## Impact

Users can be prevented from using the free Snow earning path during the farming period. The attacker can repeatedly reset the global timer and force users to wait longer or buy Snow instead.

## Proof of Concept

An existing PoC is available in `2025-06-snowman-merkle-airdrop/test/AuditPoC.t.sol`:

```solidity
function testZeroAmountBuySnowBlocksEarnSnowForFree() public {
    vm.prank(attacker);
    snow.buySnow(0);

    vm.prank(victim);
    vm.expectRevert(Snow.S__Timer.selector);
    snow.earnSnow();
}
```

Reproduction steps:

1. Attacker calls `buySnow(0)` with no ETH.
2. The call succeeds and sets `s_earnTimer = block.timestamp`.
3. A victim calls `earnSnow()`.
4. The victim's call reverts with `S__Timer`.

## Recommended Mitigation

Reject zero-amount buys and do not update the free-earn timer inside `buySnow()`.

```diff
  function buySnow(uint256 amount) external payable canFarmSnow {
+     if (amount == 0) {
+         revert S__ZeroValue();
+     }

      if (msg.value == (s_buyFee * amount)) {
          _mint(msg.sender, amount);
      } else {
          i_weth.safeTransferFrom(msg.sender, address(this), (s_buyFee * amount));
          _mint(msg.sender, amount);
      }

-     s_earnTimer = block.timestamp;
```

If the intended behavior is one free earn per user per week, replace the global timer with a per-user mapping.

```solidity
mapping(address => uint256) private s_lastEarned;
```

## Learning Notes

This finding taught me to check whether state variables are global or per-user. A shared cooldown can create denial-of-service behavior when unrelated users update the same timer.

Relevant concepts: state validation, global state, cooldown logic, denial of service.

## Analysis Disclosure

Solution-assisted learning.

This issue was discussed during review and later matched to the post-results missed finding category.
