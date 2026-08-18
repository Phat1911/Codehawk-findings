# H-01: Anyone can mint unlimited Snowman NFTs due to missing access control

## Summary

`Snowman.mintSnowman()` can be called by any address. This allows an attacker to mint arbitrary Snowman NFTs without going through the `SnowmanAirdrop` contract, without staking Snow, and without passing Merkle proof or signature checks.

## Vulnerability Details

The intended assumption is that Snowman NFTs are distributed only through valid airdrop claims. In the normal flow, `SnowmanAirdrop.claimSnowman()` verifies the receiver, signature, Merkle proof, and Snow balance before calling the NFT contract to mint.

That assumption is violated because `Snowman.mintSnowman()` is an unrestricted external function:

```solidity
function mintSnowman(address receiver, uint256 amount) external {
    for (uint256 i = 0; i < amount; i++) {
        _safeMint(receiver, s_TokenCounter);
        emit SnowmanMinted(receiver, s_TokenCounter);
        s_TokenCounter++;
    }
}
```

The problem occurs in `src/Snowman.sol` inside `mintSnowman(address receiver, uint256 amount)`. Since there is no `onlyOwner`, minter role, or trusted airdrop-contract check, any caller can directly mint NFTs.

This is unsafe because the airdrop contract's security checks do not matter when the NFT contract exposes the final minting action publicly.

## Impact

Any address can mint unlimited Snowman NFTs for free. This breaks the core NFT accounting assumption that each Snowman NFT represents a valid Snow-backed airdrop claim.

The airdrop distribution can be bypassed completely, and the NFT collection supply can be inflated by unauthorized users.

## Proof of Concept

An existing PoC is available in `2025-06-snowman-merkle-airdrop/test/AuditPoC.t.sol`:

```solidity
function testAnyoneCanMintSnowmanWithoutAirdropOrSnow() public {
    Snowman nft = new Snowman("data:image/svg+xml;base64,");
    address attacker = makeAddr("attacker");

    vm.prank(attacker);
    nft.mintSnowman(attacker, 10);

    assertEq(nft.balanceOf(attacker), 10);
}
```

Reproduction steps:

1. Deploy a `Snowman` NFT contract.
2. Use an arbitrary attacker address.
3. Call `mintSnowman(attacker, 10)` directly from the attacker.
4. Observe that the attacker receives 10 Snowman NFTs without interacting with `SnowmanAirdrop`.

## Recommended Mitigation

Restrict `mintSnowman()` so only an authorized minter can call it. The smallest reasonable fix is to allow only the trusted `SnowmanAirdrop` contract to mint.

One possible approach:

```diff
+ address private immutable i_airdrop;

  function mintSnowman(address receiver, uint256 amount) external {
+     if (msg.sender != i_airdrop) revert SM__NotAllowed();
      for (uint256 i = 0; i < amount; i++) {
          _safeMint(receiver, s_TokenCounter);
          emit SnowmanMinted(receiver, s_TokenCounter);
          s_TokenCounter++;
      }
  }
```

The exact deployment flow would also need to set `i_airdrop` to the trusted airdrop contract address. A role-based minter pattern would also work if only the airdrop contract receives the minting role.

## Learning Notes

This finding taught me the importance of access control around privileged token operations. Even if an upstream contract performs correct validation, the system can still fail when the lower-level token contract exposes sensitive functions directly.

Relevant concepts: access control, ERC721 mint authorization, protocol invariants, NFT accounting.

## Analysis Disclosure

Solution-assisted learning.

This finding was documented as part of my first CodeHawks learning workflow with assistance in structuring and validating the report.
