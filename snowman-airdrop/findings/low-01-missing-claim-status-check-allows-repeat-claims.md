# L-01: Claimed users can repeatedly claim because claim status is never checked

## Summary

`SnowmanAirdrop` records that a receiver has claimed, but never checks that record before processing another claim. A valid receiver can claim again after regaining the same Snow balance used in their Merkle leaf.

## Vulnerability Details

The intended assumption is that each eligible receiver can claim only once.

The contract appears to support that assumption with this mapping:

```solidity
mapping(address => bool) private s_hasClaimedSnowman;
```

However, `claimSnowman()` only writes to the mapping after a successful claim:

```solidity
s_hasClaimedSnowman[receiver] = true;
```

It never checks whether `s_hasClaimedSnowman[receiver]` is already true before continuing.

The problem occurs in `src/SnowmanAirdrop.sol` inside `claimSnowman()`. Since the Merkle leaf is based on the receiver's current Snow balance, a receiver included as `(receiver, 1)` can claim, later regain 1 Snow, and make the same leaf valid again.

## Impact

An eligible receiver can claim more Snowman NFTs than intended. This inflates the airdrop distribution beyond the expected one-time claim behavior.

## Proof of Concept

An existing PoC is available in `2025-06-snowman-merkle-airdrop/test/AuditPoC.t.sol`:

```solidity
function testClaimCanBeRepeatedAfterRegainingSameSnowBalance() public {
    airdrop.claimSnowman(alice, AL_PROOF, v, r, s);

    vm.warp(block.timestamp + 1 weeks);
    vm.prank(alice);
    snow.earnSnow();

    airdrop.claimSnowman(alice, AL_PROOF, v2, r2, s2);

    assertEq(nft.balanceOf(alice), 2);
}
```

Reproduction steps:

1. Alice claims with her valid proof while holding 1 Snow.
2. The airdrop contract records `s_hasClaimedSnowman[alice] = true`.
3. Alice earns or buys 1 Snow again.
4. Alice signs a new digest for the current balance.
5. The same Merkle proof for `(Alice, 1)` works again.
6. Alice receives another Snowman NFT.

## Recommended Mitigation

Check the claim status before processing the claim.

```diff
+ error SA__AlreadyClaimed();

  function claimSnowman(address receiver, bytes32[] calldata merkleProof, uint8 v, bytes32 r, bytes32 s)
      external
      nonReentrant
  {
      if (receiver == address(0)) {
          revert SA__ZeroAddress();
      }

+     if (s_hasClaimedSnowman[receiver]) {
+         revert SA__AlreadyClaimed();
+     }
```

This removes the repeat-claim path by enforcing the one-time claim invariant before signature, proof, transfer, and minting logic.

## Learning Notes

This finding taught me that recording state is not enough; the state must also be checked at the correct point in the control flow.

Relevant concepts: state validation, claim accounting, replay-like behavior, Merkle airdrops.

## Analysis Disclosure

Independent + solution verification.

This finding was submitted and matched in the CodeHawks results, with final severity shown as Low.
