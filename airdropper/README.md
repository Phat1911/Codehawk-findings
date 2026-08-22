# Airdropper - Security Review

## Challenge

Airdropper is a CodeHawks First Flights challenge about a Merkle-proof based USDC airdrop on zkSync. Four eligible Ethereum L1 addresses should each be able to claim 25 USDC, while the contract owner can withdraw the native ETH fees paid during claims.

Source repository referenced by the challenge README: `https://github.com/cyfrin/2024-04-airdropper`

## Scope

The reviewed files are:

- `src/MerkleAirdrop.sol`
- `script/Deploy.s.sol`

## Context

This review was completed as part of my CodeHawks learning practice. Some findings were identified during my own review, and missed findings were added after comparing against challenge feedback/results.

This review is not presented as an independent professional audit. It is part of a learning portfolio.

Analysis Type: Mixed - independent review + missed-findings learning

## Findings Summary

| ID | Severity | Title | Status |
| --- | --- | --- | --- |
| H-01 | High | Wrong USDC token address in `Deploy.s.sol` causes claims to fail | Missed finding |
| H-02 | High | Eligible users can replay the same Merkle proof to drain the airdrop | Independent review |
| H-03 | High | Deploy script funds 100 USDC but uses a Merkle root generated for `25e18`-unit claims | Independent review |
| H-04 | High | Account abstraction can make eligible L1 addresses unable to receive zkSync airdrops | Missed finding |
