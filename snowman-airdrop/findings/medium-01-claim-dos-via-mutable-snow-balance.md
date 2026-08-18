# M-01: User claim can be DoS'd because Merkle leaf uses mutable Snow balance

## Summary

`SnowmanAirdrop.claimSnowman()` builds the Merkle leaf using the receiver's current Snow balance. Since Snow is an ERC20 token, another user can transfer Snow to the receiver before they claim, changing the computed leaf and making the receiver's valid proof fail.

## Vulnerability Details

The intended assumption is that a user's Merkle proof remains valid for their allocated claim amount.

That assumption is violated because the contract does not take a fixed claim amount as input. Instead, it derives `amount` from mutable token state:

```solidity
uint256 amount = i_snow.balanceOf(receiver);
bytes32 leaf = keccak256(bytes.concat(keccak256(abi.encode(receiver, amount))));
```

The problem occurs in `src/SnowmanAirdrop.sol` inside `claimSnowman()`.

This is unsafe because ERC20 balances can be changed by transfers from other users. The receiver cannot reject a normal ERC20 transfer. If the Merkle tree contains `(receiver, 1)` but an attacker sends the receiver one extra Snow token, the contract computes the leaf for `(receiver, 2)` instead. The original proof no longer matches `i_merkleRoot`.

## Impact

An eligible receiver can be prevented from claiming their Snowman NFT. The attacker does not need to forge a proof or signature; they only need to change the receiver's live Snow balance before the claim.

## Proof of Concept

Minimal reproduction:

```solidity
function testAttackerCanDosClaimBySendingExtraSnow() public {
    vm.warp(block.timestamp + 1 weeks);
    vm.prank(attacker);
    snow.earnSnow();

    vm.prank(attacker);
    snow.transfer(alice, 1);

    bytes32 digest = airdrop.getMessageHash(alice);
    (uint8 v, bytes32 r, bytes32 s) = vm.sign(alKey, digest);

    vm.expectRevert(SnowmanAirdrop.SA__InvalidProof.selector);
    airdrop.claimSnowman(alice, AL_PROOF, v, r, s);
}
```

Reproduction steps:

1. Alice is included in the Merkle tree with allocation `1`.
2. An attacker obtains 1 Snow.
3. The attacker transfers 1 Snow to Alice before Alice claims.
4. Alice's balance becomes `2`.
5. `claimSnowman()` computes the leaf for `(Alice, 2)`.
6. Alice's proof for `(Alice, 1)` fails.

## Recommended Mitigation

Pass the fixed claim amount as an explicit parameter and use that value for both signature verification and Merkle proof verification.

```diff
- function claimSnowman(address receiver, bytes32[] calldata merkleProof, uint8 v, bytes32 r, bytes32 s)
+ function claimSnowman(address receiver, uint256 amount, bytes32[] calldata merkleProof, uint8 v, bytes32 r, bytes32 s)
```

Then build the leaf from the provided allocation:

```solidity
bytes32 leaf = keccak256(bytes.concat(keccak256(abi.encode(receiver, amount))));
```

The signature helper should also sign and verify the same fixed `amount`, not `balanceOf(receiver)`.

## Learning Notes

This finding taught me that Merkle proof data should be stable. Building leaves from mutable state can make valid proofs fail or create unexpected behavior.

Relevant concepts: Merkle proof validation, ERC20 transfer behavior, mutable state, denial of service.

## Analysis Disclosure

Solution-assisted learning.

This finding was reviewed after seeing the challenge results and then explained as a post-results learning note.
