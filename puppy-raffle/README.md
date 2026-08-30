# Puppy Raffle Findings

Learning write-ups for CodeHawks First Flight #2: Puppy Raffle.

Source repository referenced by the challenge README: `https://github.com/CodeHawks-Contests/ai-puppy-raffle`

Scope reviewed:

```text
2023-10-Puppy-Raffle/src/PuppyRaffle.sol
```

## Findings

| ID | Severity | Title |
| --- | --- | --- |
| H-01 | High | Reentrancy in `refund` allows an entrant to drain raffle funds |
| H-02 | High | Weak randomness lets callers and block producers bias winner selection |
| H-03 | High | Refunded players are replaced with `address(0)`, permanently DoSing raffle entry and winner selection |
| H-04 | High | Front-running winner selection with `refund` lets attackers avoid losses and steal owner fees |
| M-01 | Medium | Quadratic duplicate checks can DoS raffle entry |
| M-03 | Medium | Solidity `<0.8.0` arithmetic and `uint64` fee accounting can overflow and permanently lock fees |
| M-04 | Medium | Forced ETH can permanently break the strict balance check in `withdrawFees` |
| M-05 | Medium | Winner contracts that cannot receive ETH can DoS winner selection |

Analysis type: practice review and submission drafting.
