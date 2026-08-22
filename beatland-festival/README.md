# Beatland Festival - Security Review

## Challenge

Beatland Festival is a CodeHawks First Flights challenge about a festival NFT ecosystem. Users buy ERC1155 festival passes, attend performances to earn BEAT ERC20 rewards, and redeem ERC1155 memorabilia NFTs using those rewards.

Source repository referenced by the challenge README: `https://github.com/CodeHawks-Contests/2025-07-beatland-festival`

## Scope

The reviewed Solidity contracts are:

- `src/BeatToken.sol`
- `src/FestivalPass.sol`
- `src/Interfaces/IFestivalPass.sol`

## Context

This review was completed as part of my CodeHawks learning practice. Some findings were identified during my own review, and some were refined after challenge feedback/results.

This review is not presented as an independent professional audit. It is part of a learning portfolio.

Analysis Type: Mixed - independent review + challenge learning

## Findings Summary

| ID | Severity | Title | Status |
| --- | --- | --- | --- |
| H-01 | High | Transferable passes allow one pass to earn rewards for many addresses in the same performance | Independent review |
| H-02 | High | ERC1155 receiver reentrancy in `buyPass()` can bypass pass max supply | Challenge learning |
| H-03 | High | Memorabilia collections can never mint their full configured max supply | Independent review |
| M-01 | Medium | `configurePass()` resets sold supply while existing passes remain minted | Independent review |
| L-01 | Low | Unbounded nested loops can make `getUserMemorabiliaDetailed()` unusable | Independent review, downgraded |
| L-02 | Low | `encodeTokenId()` and `decodeTokenId()` are inconsistent for large inputs | Independent review |
