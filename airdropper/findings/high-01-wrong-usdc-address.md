# H-01: Wrong USDC token address in `Deploy.s.sol` causes claims to fail

## Summary

`Deploy.s.sol` deploys `MerkleAirdrop` with one USDC token address but funds the deployed airdrop using a different hardcoded token address. The two addresses differ by one character. As a result, the airdrop can hold one token while `claim()` attempts to transfer another token, causing valid claims to fail.

## Vulnerability Details

The deployment script stores the token used by `MerkleAirdrop`:

```solidity
address public s_zkSyncUSDC = 0x1D17CbCf0D6d143135be902365d2e5E2a16538d4;
```

That address is passed to the airdrop constructor:

```solidity
MerkleAirdrop airdrop = deployMerkleDropper(s_merkleRoot, IERC20(s_zkSyncUSDC));
```

However, the funding transfer uses a different hardcoded address:

```solidity
IERC20(0x1d17CBcF0D6D143135aE902365D2E5e2A16538D4).transfer(address(airdrop), s_amountToAirdrop);
```

The difference is:

```text
constructor token: 0x1d17cbcf0d6d143135be902365d2e5e2a16538d4
funding token:     0x1d17cbcf0d6d143135ae902365d2e5e2a16538d4
                                      ^ different nibble
```

`MerkleAirdrop.claim()` transfers the immutable token stored during construction:

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

So even if the deploy script successfully transfers funds to the airdrop contract, those funds can be for a different token than `i_airdropToken`.

## Impact

The airdrop can be deployed and funded but still be unable to process valid claims. Eligible users cannot receive the intended USDC because `claim()` attempts to transfer the token configured in the constructor, not necessarily the token actually sent to the contract.

This breaks the main airdrop flow and can leave the funded tokens stuck in the airdrop contract.

## Proof of Concept

```solidity
function testDifferentTokenAddressesBreakClaims() public {
    AirdropToken constructorToken = new AirdropToken();
    AirdropToken fundingToken = new AirdropToken();

    MerkleAirdrop airdrop = new MerkleAirdrop(merkleRoot, constructorToken);

    uint256 amountToCollect = 25 * 1e6;
    uint256 amountToSend = amountToCollect * 4;
    fundingToken.mint(address(this), amountToSend);
    fundingToken.transfer(address(airdrop), amountToSend);

    assertEq(fundingToken.balanceOf(address(airdrop)), amountToSend);
    assertEq(constructorToken.balanceOf(address(airdrop)), 0);

    vm.deal(collectorOne, airdrop.getFee());
    vm.prank(collectorOne);
    vm.expectRevert();
    airdrop.claim{value: airdrop.getFee()}(collectorOne, amountToCollect, proof);
}
```

The test models the deploy script mistake: the contract is funded with one token, but the airdrop is configured to pay another.

## Recommended Mitigation

Use the same token variable for both construction and funding. Avoid repeating hardcoded addresses.

```diff
 function run() public {
     vm.startBroadcast();
     MerkleAirdrop airdrop = deployMerkleDropper(s_merkleRoot, IERC20(s_zkSyncUSDC));
     // Send USDC -> Merkle Air Dropper
-    IERC20(0x1d17CBcF0D6D143135aE902365D2E5e2A16538D4).transfer(address(airdrop), s_amountToAirdrop);
+    IERC20(s_zkSyncUSDC).transfer(address(airdrop), s_amountToAirdrop);
     vm.stopBroadcast();
 }
```

Also verify the token address against zkSync's official USDC deployment before broadcasting.

## Learning Notes

Deployment bugs can be as severe as contract bugs. Reusing a single source of truth for critical addresses reduces the chance of funding and accounting different assets.

Relevant concepts: deployment configuration, hardcoded addresses, token funding mismatch.

## Analysis Disclosure

Missed finding.
