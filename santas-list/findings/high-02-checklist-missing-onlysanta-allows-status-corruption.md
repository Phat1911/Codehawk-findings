Title: Missing access control on checkList lets anyone corrupt Santa's first check and block users from collecting presents
Impact: High
Likelihood: High
Scope: SantasList.sol

# Root + Impact

## Description

Only Santa is supposed to update the naughty-or-nice lists. The README explicitly lists `checkList` and `checkTwice` as Santa-only actions, and `checkTwice` enforces this with `onlySanta`.

`checkList` is missing the `onlySanta` modifier. Any address can overwrite the first list entry for any user at any time. Because `checkTwice()` requires the first list value to match the second check, and `collectPresent()` also requires both stored values to match, an attacker can prevent legitimate users from being checked twice or from collecting after Santa has already approved them.

```solidity
function checkList(address person, Status status) external {
    // @> Missing onlySanta allows anyone to overwrite this status
    s_theListCheckedOnce[person] = status;
    emit CheckedOnce(person, status);
}

function checkTwice(address person, Status status) external onlySanta {
    if (s_theListCheckedOnce[person] != status) {
        revert SantasList__SecondCheckDoesntMatchFirst();
    }
    s_theListCheckedTwice[person] = status;
    emit CheckedTwice(person, status);
}
```

## Risk

**Likelihood**:

* This occurs whenever an attacker calls `checkList(victim, wrongStatus)` before Santa calls `checkTwice`.

* This also occurs after Santa has checked a user twice, because `collectPresent()` continues to read both mappings at claim time.

**Impact**:

* Legitimate NICE or EXTRA_NICE users can be blocked from collecting their NFT.

* EXTRA_NICE users can be blocked from receiving SantaToken rewards.

* The protocol's central Santa-only list integrity assumption is broken.

## Proof of Concept

Add this test to `test/unit/SantasListTest.t.sol`:

```solidity
function testAnyoneCanOverwriteFirstCheckAndBlockCollection() public {
    address victim = makeAddr("victim");
    address attacker = makeAddr("attacker");

    vm.prank(santa);
    santasList.checkList(victim, SantasList.Status.EXTRA_NICE);

    vm.prank(santa);
    santasList.checkTwice(victim, SantasList.Status.EXTRA_NICE);

    vm.prank(attacker);
    santasList.checkList(victim, SantasList.Status.NAUGHTY);

    vm.warp(santasList.CHRISTMAS_2023_BLOCK_TIME());

    vm.prank(victim);
    vm.expectRevert(SantasList.SantasList__NotNice.selector);
    santasList.collectPresent();
}
```

Santa approves the victim as `EXTRA_NICE`, but any attacker can overwrite the first check to `NAUGHTY`. The victim then fails the collection check even though Santa had already completed both checks.

## Recommended Mitigation

Apply `onlySanta` to `checkList`.

```diff
-function checkList(address person, Status status) external {
+function checkList(address person, Status status) external onlySanta {
     s_theListCheckedOnce[person] = status;
     emit CheckedOnce(person, status);
 }
```
