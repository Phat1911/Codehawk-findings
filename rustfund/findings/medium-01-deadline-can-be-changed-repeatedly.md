Title: Creator can repeatedly change a supposedly fixed deadline
Impact: Medium
Likelihood: High
Scope: programs/rustfund/src/lib.rs

# Root + Impact

## Description

Campaign deadlines are intended to define when contributions stop and when contributors may become eligible for refunds. The program contains a `dealine_set` flag and a `DeadlineAlreadySet` error, showing that deadlines are intended to be set only once.

However, `set_deadline` checks the flag but never sets it to `true`. As a result, the creator can call `set_deadline` repeatedly and move the refund/close timing after contributors have already deposited SOL.

```rust
pub fn fund_create(ctx: Context<FundCreate>, name: String, description: String, goal: u64) -> Result<()> {
    // ...
@>  fund.dealine_set = false;
    Ok(())
}

pub fn set_deadline(ctx: Context<FundSetDeadline>, deadline: u64) -> Result<()> {
    let fund = &mut ctx.accounts.fund;
@>  if fund.dealine_set {
        return Err(ErrorCode::DeadlineAlreadySet.into());
    }

@>  fund.deadline = deadline;
@>  // Missing: fund.dealine_set = true;
    Ok(())
}
```

## Risk

**Likelihood**:

* Every fund starts with `dealine_set = false`.

* Every successful call to `set_deadline` leaves `dealine_set` unchanged, so the protection never activates.

**Impact**:

* A creator can extend a failing campaign's deadline to delay contributor refunds.

* Contributors cannot rely on the deadline visible when they contribute because the creator can later change it.

## Proof of Concept

```typescript
// 1. Creator sets an initial deadline.
await program.methods.setDeadline(new anchor.BN(now + 60)).accounts(...).rpc();

// 2. Contributor sends SOL while relying on that deadline.
await program.methods.contribute(new anchor.BN(500_000_000)).accounts(...).rpc();

// 3. Creator changes the deadline again.
await program.methods.setDeadline(new anchor.BN(now + 30 * 24 * 60 * 60)).accounts(...).rpc();

// 4. Refund is blocked until the new later deadline.
await expect(program.methods.refund().accounts(...).rpc()).to.be.rejected;
```

The second `setDeadline` call succeeds because `dealine_set` is still false.

## Recommended Mitigation

Set the flag after setting the deadline. Also consider rejecting deadlines in the past or zero deadlines, depending on the intended campaign lifecycle.

```diff
 pub fn set_deadline(ctx: Context<FundSetDeadline>, deadline: u64) -> Result<()> {
     let fund = &mut ctx.accounts.fund;
     if fund.dealine_set {
         return Err(ErrorCode::DeadlineAlreadySet.into());
     }

     fund.deadline = deadline;
+    fund.dealine_set = true;
     Ok(())
 }
```
