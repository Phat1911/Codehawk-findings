# H-04: Account abstraction can make eligible L1 addresses unable to receive zkSync airdrops

## Summary

The airdrop selects eligible addresses based on Ethereum L1 activity but distributes tokens on zkSync. For EOAs, the same private key can usually control the same address on both chains. For smart contract wallets or account-abstraction accounts, an Ethereum L1 address may not be controllable at the same address on zkSync. Tokens sent to that raw address on zkSync can become inaccessible.

## Vulnerability Details

The README states that the airdrop is based on Ethereum L1 activity and will be deployed on zkSync:

```text
Our team is looking to airdrop 100 USDC tokens on the zkSync era chain to 4 lucky addresses based on their activity on the Ethereum L1.
```

The contract transfers directly to the `account` encoded in the Merkle leaf:

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

This assumes the eligible L1 `account` is also a usable recipient on zkSync. That assumption does not hold for every account type. A Safe, smart contract wallet, or other account-abstraction address on Ethereum L1 does not automatically have a private key or equivalent deployed control at the same address on zkSync.

The contract also provides no way for the eligible L1 account to authorize a different zkSync recipient.

## Impact

Eligible users whose L1 activity belongs to a smart contract wallet can be unable to access their airdropped zkSync USDC. The tokens may be transferred to an address that looks correct but is not controlled by the user on zkSync.

This can permanently deny legitimate recipients their airdrop.

## Proof of Concept

Consider this eligible entry:

```text
eligible L1 account: 0xSafeOnEthereum
amount: 25 USDC
```

The Merkle leaf is built from:

```solidity
keccak256(bytes.concat(keccak256(abi.encode(0xSafeOnEthereum, amount))));
```

When claimed on zkSync, the contract executes:

```solidity
i_airdropToken.safeTransfer(0xSafeOnEthereum, amount);
```

For an EOA, the same private key can control that address on zkSync. For an L1 smart contract wallet, there is no private key for `0xSafeOnEthereum`, and the same wallet may not be deployed or controllable at that zkSync address. The airdropped funds are therefore sent to an unusable recipient address.

## Recommended Mitigation

Allow eligible L1 accounts to authorize a separate zkSync recipient address.

One approach is to keep the Merkle proof over the eligible L1 account and amount, then require a signature authorizing the destination recipient:

```diff
-function claim(address account, uint256 amount, bytes32[] calldata merkleProof) external payable {
+function claim(
+    address account,
+    address recipient,
+    uint256 amount,
+    bytes32[] calldata merkleProof,
+    bytes calldata authorization
+) external payable {
     if (msg.value != FEE) {
         revert MerkleAirdrop__InvalidFeeAmount();
     }
     bytes32 leaf = keccak256(bytes.concat(keccak256(abi.encode(account, amount))));
     if (!MerkleProof.verify(merkleProof, i_merkleRoot, leaf)) {
         revert MerkleAirdrop__InvalidProof();
     }
+    // Verify account authorized recipient for this amount.
+    // For smart contract wallets, support ERC-1271 signature validation.
     emit Claimed(account, amount);
-    i_airdropToken.safeTransfer(account, amount);
+    i_airdropToken.safeTransfer(recipient, amount);
 }
```

Alternatively, collect zkSync recipient addresses before generating the Merkle tree and include the destination address in each leaf:

```solidity
keccak256(abi.encode(l1EligibleAccount, zkSyncRecipient, amount))
```

Deploying the airdrop on Ethereum L1 is another valid mitigation when eligibility is tied directly to L1 account addresses.

## Learning Notes

Cross-chain airdrops should not assume every source-chain address is a controllable destination-chain recipient. Smart contract wallets and account abstraction make explicit recipient authorization important.

Relevant concepts: account abstraction, smart contract wallets, cross-chain address assumptions, ERC-1271.

## Analysis Disclosure

Missed finding.
