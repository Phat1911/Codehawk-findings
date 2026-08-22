# DatingDapp - Security Review

## Challenge

DatingDapp is a CodeHawks First Flights challenge about a dating protocol where users mint soulbound NFT profiles, pay ETH to like other users, and receive pooled match funds in a shared multisig wallet when a like is mutual.

Source repository referenced by the challenge README: `https://github.com/CodeHawks-Contests/2025-02-datingdapp`

## Scope

The reviewed Solidity contracts are:

- `src/SoulboundProfileNFT.sol`
- `src/LikeRegistry.sol`
- `src/MultiSig.sol`

## Context

This review was completed as part of my CodeHawks learning practice. Some findings were identified during my own review, and missed findings were added after comparing against challenge feedback/results.

This review is not presented as an independent professional audit. It is part of a learning portfolio.

Analysis Type: Mixed - independent review + missed-findings learning

## Findings Summary

| ID | Severity | Title | Status |
| --- | --- | --- | --- |
| H-01 | High | Paid like ETH is never credited, causing zero rewards and stuck funds | Independent review |
| M-01 | Medium | Blocked users can mint a new profile and bypass moderation | Independent review |
| M-02 | Medium | `matchRewards()` can let users receive later dates without contributing funds | Missed finding |
| M-03 | Medium | App owner can lock users' deposited funds by blocking profiles | Missed finding |
| M-04 | Medium | Reentrancy in `mintProfile()` allows multiple soulbound NFTs per address | Missed finding |
