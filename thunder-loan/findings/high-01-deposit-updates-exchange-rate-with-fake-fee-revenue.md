# H-01: `deposit()` updates the exchange rate with fake fee revenue

## Summary

`ThunderLoan.deposit()` calculates a flash loan fee from the deposit amount and updates the AssetToken exchange rate even though the depositor only transfers the principal amount. This creates fake yield that was never paid into the vault.

## Vulnerability Details

The intended behavior is that LPs deposit underlying tokens and receive AssetToken shares. Deposits should not create fee revenue. Only completed flash loans should increase the exchange rate because only flash loans generate fees for existing LPs.

In `ThunderLoan.deposit()`, shares are minted, then the protocol calculates a fee from the deposit amount and calls `updateExchangeRate()`:

```solidity
function deposit(IERC20 token, uint256 amount) external revertIfZero(amount) revertIfNotAllowedToken(token) {
    AssetToken assetToken = s_tokenToAssetToken[token];
    uint256 exchangeRate = assetToken.getExchangeRate();
    uint256 mintAmount = (amount * assetToken.EXCHANGE_RATE_PRECISION()) / exchangeRate;
    emit Deposit(msg.sender, token, amount);
    assetToken.mint(msg.sender, mintAmount);

    uint256 calculatedFee = getCalculatedFee(token, amount);
    assetToken.updateExchangeRate(calculatedFee);

    token.safeTransferFrom(msg.sender, address(assetToken), amount);
}
```

The depositor transfers only `amount`, not `amount + calculatedFee`. Therefore, the exchange rate is increased without matching underlying assets being added to the AssetToken vault.

## Impact

The exchange rate becomes higher than the vault's real backing. Later depositors receive fewer shares than they should, and full redemptions can revert because shares claim more underlying than the vault actually holds.

This breaks the central LP accounting invariant: total redeemable underlying should not exceed the underlying held by the AssetToken.

## Proof of Concept

An existing PoC was added in `2023-11-Thunder-Loan/test/unit/ThunderLoanFindingsPoC.t.sol`:

```solidity
function testDepositCreatesFakeYieldAndBreaksRedeem() public {
    ThunderLoan thunderLoan = _deployThunderLoan();
    thunderLoan.setAllowedToken(tokenA, true);
    AssetToken assetToken = thunderLoan.getAssetFromToken(tokenA);

    address lp = address(1);
    uint256 amount = 1000e18;
    tokenA.mint(lp, amount);

    vm.startPrank(lp);
    tokenA.approve(address(thunderLoan), amount);
    thunderLoan.deposit(tokenA, amount);

    assertGt(assetToken.getExchangeRate(), assetToken.EXCHANGE_RATE_PRECISION());

    vm.expectRevert();
    thunderLoan.redeem(tokenA, type(uint256).max);
    vm.stopPrank();
}
```

Reproduction steps:

1. Deploy `ThunderLoan` behind the proxy.
2. Allow `tokenA`.
3. Have an LP deposit `1000e18` tokenA.
4. Observe that the exchange rate increased even though no flash loan fee was paid.
5. Attempt to redeem all shares.
6. The redeem reverts because the inflated exchange rate claims more underlying than the vault holds.

## Recommended Mitigation

Remove the fee calculation and exchange rate update from `deposit()`:

```diff
function deposit(IERC20 token, uint256 amount) external revertIfZero(amount) revertIfNotAllowedToken(token) {
    AssetToken assetToken = s_tokenToAssetToken[token];
    uint256 exchangeRate = assetToken.getExchangeRate();
    uint256 mintAmount = (amount * assetToken.EXCHANGE_RATE_PRECISION()) / exchangeRate;
    emit Deposit(msg.sender, token, amount);
    assetToken.mint(msg.sender, mintAmount);
-   uint256 calculatedFee = getCalculatedFee(token, amount);
-   assetToken.updateExchangeRate(calculatedFee);
    token.safeTransferFrom(msg.sender, address(assetToken), amount);
}
```

Only call `updateExchangeRate()` after real flash loan fee revenue has been repaid to the vault.

## Learning Notes

This finding shows why share price changes must be backed by real asset movements. Internal accounting cannot safely assume a fee exists unless the contract has actually received it.

Relevant concepts: vault accounting, share price inflation, exchange rates, accounting invariants.

## Analysis Disclosure

Guided independent review.
