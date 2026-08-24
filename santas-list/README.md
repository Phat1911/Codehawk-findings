# Santas List - Security Review

## Challenge

Santas List is a CodeHawks First Flights challenge about an on-chain naughty-or-nice list. Santa checks users twice, eligible users collect ERC721 present NFTs, and EXTRA_NICE users receive SantaTokens that can be used to buy additional presents.

Source repository referenced by the challenge README: `https://github.com/Cyfrin/2023-11-Santas-List`

## Scope

The reviewed Solidity contracts are:

- `src/SantasList.sol`
- `src/SantaToken.sol`
- `src/TokenUri.sol`

## Context

This review was completed as part of my CodeHawks learning practice. Findings were identified during review and refined after comparing wording with accepted challenge findings.

This review is not presented as an independent professional audit. It is part of a learning portfolio.

Analysis Type: Mixed - independent review + challenge feedback learning

## Findings Summary

| ID | Severity | Title | Status |
| --- | --- | --- | --- |
| H-01 | High | `buyPresent()` burns tokens from `presentReceiver` but mints the NFT to the caller | Independent review, wording refined |
| H-02 | High | Missing access control on `checkList()` lets anyone corrupt Santa's first check | Independent review |
| H-03 | High | Unchecked users are treated as `NICE` by default and can collect presents | Independent review |
| H-04 | High | Transferable present NFTs let users bypass the one-present-per-address collection limit | Independent review |
| H-05 | High | `buyPresent()` charges `1e18` SantaToken despite the documented `2e18` purchase cost | Independent review |
| M-01 | Medium | Test suite executes arbitrary local commands through Foundry FFI | Repository/test-suite security note |
