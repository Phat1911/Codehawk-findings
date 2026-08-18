# H-02: Inconsistent `MESSAGE_TYPEHASH` breaks standard EIP-712 signatures

## Summary

`SnowmanAirdrop` uses an incorrect EIP-712 type string for `SnowmanClaim`. Because EIP-712 hashes are exact, standard wallets or frontends that sign the correct typed data will produce signatures that the contract rejects.

## Vulnerability Details

The intended assumption is that a receiver can sign a standard EIP-712 `SnowmanClaim` message authorizing someone else to claim on their behalf.

The struct is:

```solidity
struct SnowmanClaim {
    address receiver;
    uint256 amount;
}
```

The standard type string for this struct should be:

```text
SnowmanClaim(address receiver,uint256 amount)
```

However, `SnowmanAirdrop` defines `MESSAGE_TYPEHASH` with `address` misspelled as `addres`:

```solidity
keccak256("SnowmanClaim(addres receiver, uint256 amount)")
```

This problem occurs in `src/SnowmanAirdrop.sol` where `getMessageHash()` builds the digest that `_isValidSignature()` later verifies.

This is unsafe because EIP-712 signing tools construct the digest from the correct schema, while the contract verifies against a different schema. The recovered signer will not match the receiver for a normal EIP-712 signature.

## Impact

Users signing standard EIP-712 typed data can be unable to claim through the intended signature-based flow. The claim-on-behalf feature becomes incompatible with normal wallet/frontend signing behavior.

## Proof of Concept

An existing PoC is available in `2025-06-snowman-merkle-airdrop/test/TestSnowmanAirdrop.t.sol`:

```solidity
function testStandardEIP712SignatureFailsDueToInvalidTypehash() public {
    bytes32 standardMessageTypehash = keccak256("SnowmanClaim(address receiver,uint256 amount)");
    bytes32 standardStructHash = keccak256(abi.encode(standardMessageTypehash, alice, amount));
    bytes32 standardDigest = keccak256(abi.encodePacked("\x19\x01", domainSeparator, standardStructHash));
    (uint8 v, bytes32 r, bytes32 s) = vm.sign(alKey, standardDigest);

    vm.expectRevert(SnowmanAirdrop.SA__InvalidSignature.selector);
    airdrop.claimSnowman(alice, AL_PROOF, v, r, s);
}
```

Reproduction steps:

1. Build a standard EIP-712 digest for `SnowmanClaim(address receiver,uint256 amount)`.
2. Have the receiver sign that digest.
3. Submit the signature to `claimSnowman()`.
4. Observe that the claim reverts with `SA__InvalidSignature`.

## Recommended Mitigation

Use the exact standard EIP-712 type string for the `SnowmanClaim` struct.

```diff
- keccak256("SnowmanClaim(addres receiver, uint256 amount)")
+ keccak256("SnowmanClaim(address receiver,uint256 amount)")
```

Tests should include at least one independently constructed standard EIP-712 digest, rather than only signing the digest returned by `getMessageHash()`.

## Learning Notes

This finding taught me that EIP-712 type hashes are brittle by design: the type string must match exactly. A typo in the schema changes the digest and breaks signature verification.

Relevant concepts: EIP-712, typed data, signature verification, digest construction.

## Analysis Disclosure

Solution-assisted learning.

This finding was reviewed after seeing the challenge results and then reproduced with a focused local PoC.
