Title: Withdrawals leave stale amount_raised and block later withdrawals
Impact: Medium
Likelihood: Medium
Scope: programs/rustfund/src/lib.rs

# Root + Impact

## Description

`amount_raised` is used as the amount the creator can withdraw. After a withdrawal, the program transfers lamports out of the fund account but does not reduce or reset `amount_raised`.

If the creator withdraws once and new contributions arrive later, `amount_raised` contains both already-withdrawn and newly-raised amounts. A later withdrawal attempts to transfer the full lifetime total again, which can exceed the fund account's available lamports and revert. This can permanently block withdrawal of later contributions unless enough extra lamports are somehow added to cover the stale amount.

```rust
pub fn withdraw(ctx: Context<FundWithdraw>) -> Result<()> {
@>  let amount = ctx.accounts.fund.amount_raised;

    **ctx.accounts.fund.to_account_info().try_borrow_mut_lamports()? =
        ctx.accounts.fund.to_account_info().lamports()
        .checked_sub(amount)
        .ok_or(ProgramError::InsufficientFunds)?;

    **ctx.accounts.creator.to_account_info().try_borrow_mut_lamports()? =
        ctx.accounts.creator.to_account_info().lamports()
        .checked_add(amount)
        .ok_or(ErrorCode::CalculationOverflow)?;

@>  // Missing: update withdrawn/available accounting after transfer
    Ok(())
}
```

## Risk

**Likelihood**:

* This occurs when a campaign receives contributions after a prior withdrawal, because contributions remain open until the deadline logic blocks them.

* The program has no `withdrawn_amount`, `available_to_withdraw`, or finalized campaign state to separate new funds from previously withdrawn funds.

**Impact**:

* Creators can be unable to withdraw later valid contributions.

* The fund's accounting no longer represents the withdrawable lamport balance, causing denial of service for normal creator withdrawals.

## Proof of Concept

```typescript
// 1. Contributor A contributes 10 lamports.
await program.methods.contribute(new anchor.BN(10)).accounts(...).rpc();

// 2. Creator withdraws. amountRaised remains 10.
await program.methods.withdraw().accounts(...).rpc();

// 3. Contributor B contributes 5 lamports. amountRaised becomes 15.
await program.methods.contribute(new anchor.BN(5)).accounts(...).rpc();

// 4. Creator tries to withdraw again.
await program.methods.withdraw().accounts(...).rpc();
```

The second withdrawal attempts to transfer `15` lamports even though only the later `5` lamports were added after the first withdrawal. The transfer can fail with insufficient funds because `amount_raised` includes already-withdrawn value.

## Recommended Mitigation

Track withdrawable funds separately from lifetime raised funds, or close the campaign to new contributions after withdrawal. One simple accounting fix is to add a `withdrawn_amount` field and withdraw only the delta.

```diff
 pub struct Fund {
     pub goal: u64,
     pub deadline: u64,
     pub creator: Pubkey,
     pub amount_raised: u64,
+    pub withdrawn_amount: u64,
     pub dealine_set: bool,
 }

 pub fn withdraw(ctx: Context<FundWithdraw>) -> Result<()> {
-    let amount = ctx.accounts.fund.amount_raised;
+    let fund = &mut ctx.accounts.fund;
+    let amount = fund
+        .amount_raised
+        .checked_sub(fund.withdrawn_amount)
+        .ok_or(ErrorCode::CalculationOverflow)?;

     // transfer lamports...

+    fund.withdrawn_amount = fund
+        .withdrawn_amount
+        .checked_add(amount)
+        .ok_or(ErrorCode::CalculationOverflow)?;
     Ok(())
 }
```
