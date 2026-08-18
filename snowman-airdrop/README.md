# Snowman Airdrop - Security Review

## Challenge

Snowman Airdrop is a CodeHawks First Flights challenge about an ERC20 Snow token, an ERC721 Snowman NFT, and a Merkle/signature-based airdrop flow.

Original CodeHawks challenge URL: https://codehawks.cyfrin.io/c/ai-snowman-merkle-airdrop-cmslquoyw0000l904hgbq606p

Source repository referenced by the challenge README: `https://github.com/CodeHawks-Contests/2025-06-snowman-merkle-airdrop.git`

## Scope

The reviewed Solidity contracts are:

- `src/Snow.sol`
- `src/Snowman.sol`
- `src/SnowmanAirdrop.sol`

## Context

This was my first CodeHawks challenge and first security finding write-up. I completed it as a learning exercise while building familiarity with Solidity, ERC20/ERC721 behavior, Merkle proofs, EIP-712 signatures, and smart contract security review workflow.

This review is not presented as an independent professional audit. It is part of a learning portfolio.

Analysis Type: Mixed - submitted findings + solution-assisted learning

## Findings Summary

| ID | Severity | Title | Status |
| --- | --- | --- | --- |
| H-01 | High | Anyone can mint unlimited Snowman NFTs due to missing access control in `mintSnowman()` | Submitted and matched |
| H-02 | High | Inconsistent `MESSAGE_TYPEHASH` breaks standard EIP-712 signatures | Post-results learning |
| M-01 | Medium | A user's claim can be DoS'd by changing their Snow balance | Post-results learning |
| L-01 | Low | Claimed users can repeatedly claim because claim status is never checked | Submitted and matched |
| L-02 | Low | Global timer reset in `Snow::buySnow` denies free Snow claims for all users | Post-results learning |

## What I Learned

- Public/external functions that change critical state need explicit authorization.
- NFT minting is a privileged action when token supply is meant to represent eligibility or completed claims.
- A working airdrop verification flow can still be bypassed if the downstream mint function is directly callable.
- EIP-712 type strings must exactly match the intended typed data schema.
- Merkle leaves should be built from fixed claim data, not mutable token balances.
- Shared global timers can accidentally let one user's action affect everyone else.
- A strong finding should explain the intended invariant, the exact violated assumption, who can trigger it, and the concrete impact.
