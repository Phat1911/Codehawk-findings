Title: Unchecked users are treated as NICE by default and can collect presents without being checked twice
Impact: High
Likelihood: High
Scope: SantasList.sol

# Root + Impact

## Description

The intended behavior is that only users checked twice by Santa as `NICE` or `EXTRA_NICE` can collect a present. The code relies on two mappings to store those statuses.

The `Status` enum declares `NICE` as the first value. In Solidity, the default value for an unset enum is its first value, which is numeric value `0`. Unset mapping entries also return the default value. As a result, any address that has never been checked has both list values equal to `Status.NICE`.

```solidity
enum Status {
    // @> Default enum value is 0, so unset mapping entries are NICE
    NICE,
    EXTRA_NICE,
    NAUGHTY,
    NOT_CHECKED_TWICE
}

mapping(address person => Status naughtyOrNice) private s_theListCheckedOnce;
mapping(address person => Status naughtyOrNice) private s_theListCheckedTwice;

function collectPresent() external {
    ...
    if (s_theListCheckedOnce[msg.sender] == Status.NICE && s_theListCheckedTwice[msg.sender] == Status.NICE) {
        // @> Unchecked users satisfy this branch by default
        _mintAndIncrement();
        return;
    }
    ...
}
```

## Risk

**Likelihood**:

* This occurs for every address that has never been written to either status mapping.

* After Christmas, any unchecked address can directly call `collectPresent()`.

**Impact**:

* Anyone can mint a present NFT without being approved by Santa.

* The protocol's core eligibility rule, "must be checked twice", is bypassed.

## Proof of Concept

Add this test to `test/unit/SantasListTest.t.sol`:

```solidity
function testUncheckedUserCanCollectPresentBecauseDefaultStatusIsNice() public {
    address uncheckedUser = makeAddr("uncheckedUser");

    assertEq(uint256(santasList.getNaughtyOrNiceOnce(uncheckedUser)), uint256(SantasList.Status.NICE));
    assertEq(uint256(santasList.getNaughtyOrNiceTwice(uncheckedUser)), uint256(SantasList.Status.NICE));

    vm.warp(santasList.CHRISTMAS_2023_BLOCK_TIME());

    vm.prank(uncheckedUser);
    santasList.collectPresent();

    assertEq(santasList.balanceOf(uncheckedUser), 1);
}
```

The user was never checked by Santa, but both mappings return `Status.NICE` by default, allowing collection.

## Recommended Mitigation

Make the default enum value an ineligible status, and require explicit Santa updates for eligible states.

```diff
 enum Status {
+    UNKNOWN,
     NICE,
     EXTRA_NICE,
     NAUGHTY,
     NOT_CHECKED_TWICE
 }
```

Alternatively, track whether each mapping entry has been explicitly set.

```diff
+mapping(address person => bool checkedOnce) private s_hasBeenCheckedOnce;
+mapping(address person => bool checkedTwice) private s_hasBeenCheckedTwice;

 function checkList(address person, Status status) external onlySanta {
     s_theListCheckedOnce[person] = status;
+    s_hasBeenCheckedOnce[person] = true;
     emit CheckedOnce(person, status);
 }
```

Then require both explicit flags in `collectPresent()`.
