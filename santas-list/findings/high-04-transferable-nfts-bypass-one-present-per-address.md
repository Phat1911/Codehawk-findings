Title: Transferable present NFTs let users bypass the one-present-per-address collection limit
Impact: High
Likelihood: High
Scope: SantasList.sol

# Root + Impact

## Description

The intended behavior is that each address can collect only one present through `collectPresent()`. The implementation tries to enforce this by checking whether the caller currently owns any present NFT.

The contract inherits a normal transferable ERC721. A user can collect a present, transfer the NFT away, and then call `collectPresent()` again because their current `balanceOf(msg.sender)` is zero. For an `EXTRA_NICE` user, each repeated collection also mints another SantaToken.

```solidity
function collectPresent() external {
    ...
    // @> Checks current NFT balance, not whether the address has already collected
    if (balanceOf(msg.sender) > 0) {
        revert SantasList__AlreadyCollected();
    }
    ...
    } else if (
        s_theListCheckedOnce[msg.sender] == Status.EXTRA_NICE
            && s_theListCheckedTwice[msg.sender] == Status.EXTRA_NICE
    ) {
        _mintAndIncrement();
        // @> Repeated collection also repeats SantaToken minting
        i_santaToken.mint(msg.sender);
        return;
    }
}
```

## Risk

**Likelihood**:

* This occurs whenever an eligible user can transfer their ERC721 present to another address.

* ERC721 transfer functions are enabled by default because the contract does not override or restrict transfers.

**Impact**:

* A single eligible address can mint unlimited present NFTs by repeatedly transferring away the previous NFT.

* An `EXTRA_NICE` address can also mint unlimited SantaTokens, inflating the protocol reward token supply.

## Proof of Concept

Add this test to `test/unit/SantasListTest.t.sol`:

```solidity
function testUserCanTransferAwayNftAndCollectAgain() public {
    address extraNiceUser = makeAddr("extraNiceUser");
    address receiver = makeAddr("receiver");

    vm.prank(santa);
    santasList.checkList(extraNiceUser, SantasList.Status.EXTRA_NICE);

    vm.prank(santa);
    santasList.checkTwice(extraNiceUser, SantasList.Status.EXTRA_NICE);

    vm.warp(santasList.CHRISTMAS_2023_BLOCK_TIME());

    vm.prank(extraNiceUser);
    santasList.collectPresent();

    assertEq(santasList.balanceOf(extraNiceUser), 1);
    assertEq(santaToken.balanceOf(extraNiceUser), 1e18);

    vm.prank(extraNiceUser);
    santasList.transferFrom(extraNiceUser, receiver, 0);

    assertEq(santasList.balanceOf(extraNiceUser), 0);

    vm.prank(extraNiceUser);
    santasList.collectPresent();

    assertEq(santasList.balanceOf(extraNiceUser), 1);
    assertEq(santasList.balanceOf(receiver), 1);
    assertEq(santaToken.balanceOf(extraNiceUser), 2e18);
}
```

The second collection succeeds because the contract only checks current ownership, not previous collection history.

## Recommended Mitigation

Track collection history in a dedicated mapping instead of relying on the caller's current ERC721 balance.

```diff
+mapping(address user => bool collected) private s_hasCollected;

 function collectPresent() external {
     if (block.timestamp < CHRISTMAS_2023_BLOCK_TIME) {
         revert SantasList__NotChristmasYet();
     }
-    if (balanceOf(msg.sender) > 0) {
+    if (s_hasCollected[msg.sender]) {
         revert SantasList__AlreadyCollected();
     }
+    s_hasCollected[msg.sender] = true;
     if (s_theListCheckedOnce[msg.sender] == Status.NICE && s_theListCheckedTwice[msg.sender] == Status.NICE) {
         _mintAndIncrement();
         return;
```

If presents are intended to be soulbound, also override transfer functions to block transfers after mint.
