# H-02: Storage layout corruption after upgrade changes flash loan fee to 100%

## Summary

`ThunderLoanUpgraded` changes the storage layout of the original implementation by removing `s_feePrecision` and replacing it with a constant. After the proxy upgrades, the old proxy storage is interpreted using the new layout, causing `s_flashLoanFee` to read the old `s_feePrecision` value of `1e18`.

## Vulnerability Details

Upgradeable contracts must preserve storage layout across implementations. The proxy keeps the same storage slots forever; only the implementation code changes.

The original `ThunderLoan` stores `s_feePrecision` before `s_flashLoanFee`:

```solidity
mapping(IERC20 => AssetToken) public s_tokenToAssetToken;

uint256 private s_feePrecision;
uint256 private s_flashLoanFee;

mapping(IERC20 token => bool currentlyFlashLoaning) private s_currentlyFlashLoaning;
```

The upgraded implementation removes `s_feePrecision` from storage and uses a constant:

```solidity
mapping(IERC20 => AssetToken) public s_tokenToAssetToken;

uint256 private s_flashLoanFee;
uint256 public constant FEE_PRECISION = 1e18;

mapping(IERC20 token => bool currentlyFlashLoaning) private s_currentlyFlashLoaning;
```

Constants do not occupy storage slots. Therefore, after upgrade, `ThunderLoanUpgraded.s_flashLoanFee` reads the old slot that held `ThunderLoan.s_feePrecision`.

Before upgrade:

```text
s_feePrecision = 1e18
s_flashLoanFee = 3e15
```

After upgrade:

```text
s_flashLoanFee = old s_feePrecision = 1e18
```

The flash loan fee changes from `0.3%` to `100%`.

## Impact

Flash loans become incorrectly priced after the planned upgrade. Borrowers are charged a 100% fee instead of the intended 0.3% fee. Storage values after the shifted slot can also be misinterpreted, creating broader protocol instability.

This is high impact because the protocol's planned upgrade path corrupts core accounting configuration.

## Proof of Concept

An existing PoC was added in `2023-11-Thunder-Loan/test/unit/ThunderLoanFindingsPoC.t.sol`:

```solidity
function testUpgradeCorruptsFlashLoanFee() public {
    ThunderLoan implementation = new ThunderLoan();
    ERC1967Proxy proxy = new ERC1967Proxy(address(implementation), "");

    ThunderLoan thunderLoan = ThunderLoan(address(proxy));
    thunderLoan.initialize(address(mockPoolFactory));

    assertEq(thunderLoan.getFee(), 3e15);

    ThunderLoanUpgraded upgraded = new ThunderLoanUpgraded();
    thunderLoan.upgradeTo(address(upgraded));

    ThunderLoanUpgraded upgradedProxy = ThunderLoanUpgraded(address(proxy));

    assertEq(upgradedProxy.getFee(), 1e18);
}
```

The important point is that both `thunderLoan` and `upgradedProxy` are typed views of the same proxy address. The first call reads the proxy through the old ABI, and the final call reads the same proxy through the upgraded ABI.

## Recommended Mitigation

Preserve the original storage layout:

```diff
 mapping(IERC20 => AssetToken) public s_tokenToAssetToken;

+uint256 private s_feePrecision;
 uint256 private s_flashLoanFee;
-uint256 public constant FEE_PRECISION = 1e18;

 mapping(IERC20 token => bool currentlyFlashLoaning) private s_currentlyFlashLoaning;
```

Alternatively, leave `s_feePrecision` in storage and add new variables only after all existing variables. Use OpenZeppelin upgrade validation tooling to detect layout incompatibilities before deployment.

## Learning Notes

This finding shows why implementation contracts do not own the meaningful state in a proxy system. The proxy stores the data, and implementation code only defines how that data is interpreted.

Relevant concepts: UUPS proxy, storage slots, upgrade safety, layout compatibility.

## Analysis Disclosure

Guided independent review.
