# H-03: Deploy script funds 100 USDC but uses a Merkle root generated for `25e18`-unit claims

## Summary

The deployment script funds the airdrop with `4 * 25e6` USDC units, which matches USDC's 6 decimals. However, the Merkle root used by the deployment is generated from leaves containing `25e18` units per user. Valid proofs therefore request far more tokens than the airdrop contract holds, causing claims to fail.

## Vulnerability Details

The README states that four users should each receive 25 USDC. `Deploy.s.sol` funds the airdrop with 100 USDC using 6 decimals:

```solidity
// 4 users, 25 USDC each
uint256 public s_amountToAirdrop = 4 * (25 * 1e6);
```

But `makeMerkle.js`, which is used to generate the Merkle root in `Deploy.s.sol`, encodes the amount with 18 decimals:

```javascript
const amount = (25 * 1e18).toString()
```

The generated `tree.json` confirms every leaf uses:

```text
25000000000000000000
```

The airdrop verifies the claimant-provided amount against this root:

```solidity
bytes32 leaf = keccak256(bytes.concat(keccak256(abi.encode(account, amount))));
if (!MerkleProof.verify(merkleProof, i_merkleRoot, leaf)) {
    revert MerkleAirdrop__InvalidProof();
}
i_airdropToken.safeTransfer(account, amount);
```

So a valid proof is valid for `25e18`, not `25e6`. The transfer then attempts to send `25e18` USDC units from a contract funded with only `100e6` units.

## Impact

Eligible users cannot claim the intended 25 USDC because the valid Merkle amount is larger than the entire funded airdrop balance. This makes the deployed airdrop unusable.

## Proof of Concept

```solidity
function testValidMerkleAmountExceedsAirdropFunding() public {
    AirdropToken token = new AirdropToken();

    bytes32 root = 0xf69aaa25bd4dd10deb2ccd8235266f7cc815f6e9d539e9f4d47cae16e0c36a05;
    MerkleAirdrop airdrop = new MerkleAirdrop(root, token);

    uint256 deployedFunding = 4 * (25 * 1e6);
    uint256 merkleClaimAmount = 25 * 1e18;

    token.mint(address(this), deployedFunding);
    token.transfer(address(airdrop), deployedFunding);

    assertLt(token.balanceOf(address(airdrop)), merkleClaimAmount);

    vm.deal(collectorOne, airdrop.getFee());
    vm.prank(collectorOne);
    vm.expectRevert();
    airdrop.claim{value: airdrop.getFee()}(collectorOne, merkleClaimAmount, proof);
}
```

## Recommended Mitigation

Generate the Merkle tree with USDC's 6-decimal units, then update the deployment root.

```diff
-const amount = (25 * 1e18).toString()
+const amount = (25 * 1e6).toString()
```

Regenerate `tree.json` and copy the corrected root into `Deploy.s.sol`:

```diff
-bytes32 public s_merkleRoot = 0xf69aaa25bd4dd10deb2ccd8235266f7cc815f6e9d539e9f4d47cae16e0c36a05;
+bytes32 public s_merkleRoot = CORRECT_ROOT_FOR_25E6_AMOUNTS;
```

## Learning Notes

Merkle leaves are part of protocol accounting. Token decimals used in off-chain tree generation must match the deployed token and the funding amount.

Relevant concepts: token decimals, deployment scripts, Merkle leaf generation.

## Analysis Disclosure

Independent review.
