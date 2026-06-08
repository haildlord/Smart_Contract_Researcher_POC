# M-14 | `PhiNFT1155` Cannot Be Effectively Paused

**Author:** hailthelord  
**Proof Link:** https://github.com/code-423n4/2024-08-phi-findings/issues/268
**Severity:** Medium  
**Category:** Pausable  
**Type:** Incorrect Inheritance / Missing Pause Enforcement  
**Protocol:** Phi Protocol  
**Submitted via:** Code4rena  

## Summary

`PhiNFT1155` inherits from `PausableUpgradeable`, but this has **no effect** on ERC1155 transfer functions. The contract should inherit from `ERC1155PausableUpgradeable` instead. As a result, the pause mechanism is completely broken — users can still transfer NFTs even when the contract is paused.

## Vulnerability Details

### Current Inheritance (Incorrect)

```solidity
contract PhiNFT1155 is
    Initializable,
    UUPSUpgradeable,
    ERC1155SupplyUpgradeable,
    ReentrancyGuardUpgradeable,
    PausableUpgradeable,           // ← Does NOT affect ERC1155 transfers
    Ownable2StepUpgradeable,
    IPhiNFT1155,
    Claimable,
    CreatorRoyaltiesControl
{ ... }
```

`PausableUpgradeable` only provides the `paused()` state and `whenNotPaused` / `whenPaused` modifiers. It does **not** hook into ERC1155’s `_update` function.

OpenZeppelin provides `ERC1155PausableUpgradeable` specifically for this purpose. It overrides `_update` to add a pause check before any mint/transfer/burn.

### Proof of Concept (Full Code)

Drop this test into `PhiFactory.t.sol` and run with `forge test --match-test testKuprum_PhiNFT1155PauseNotWorking`:

```solidity
function testKuprum_PhiNFT1155PauseNotWorking() public {
    _createArt(ART_ID_URL_STRING);
    uint256 artId = 1;
    bytes32 advanced_data = bytes32("1");
    bytes memory signData =
        abi.encode(expiresIn, participant, referrer, verifier, artId, block.chainid, advanced_data);
    bytes32 msgHash = keccak256(signData);
    bytes32 digest = ECDSA.toEthSignedMessageHash(msgHash);
    (uint8 v, bytes32 r, bytes32 s) = vm.sign(claimSignerPrivateKey, digest);
    if (v != 27) s = s | bytes32(uint256(1) << 255);
    bytes memory signature = abi.encodePacked(r, s);
    bytes memory data =
        abi.encode(1, participant, referrer, verifier, expiresIn, uint256(1), advanced_data, IMAGE_URL, signature);
    bytes memory dataCompressed = LibZip.cdCompress(data);
    uint256 totalMintFee = phiFactory.getArtMintFee(1, 1);

    vm.startPrank(participant, participant);
    phiFactory.claim{ value: totalMintFee }(dataCompressed);

    // referrer payout
    address artAddress = phiFactory.getArtAddress(1);
    assertEq(IERC1155(artAddress).balanceOf(participant, 1), 1, "particpiant erc1155 balance");
    
    // Everything up to here is from `test_claim_1155_with_ref`
    
    // Owner pauses the art contract
    vm.startPrank(owner);
    phiFactory.pauseArtContract(artAddress);

    // Users are still able to transfer NFTs despite the contract being paused
    assertEq(IERC1155(artAddress).balanceOf(user1, 1), 0, "user1 doesn't have any tokens");
    vm.startPrank(participant);
    IERC1155(artAddress).safeTransferFrom(participant, user1, 1, 1, hex"00");
    assertEq(IERC1155(artAddress).balanceOf(participant, 1), 0, "particpiant now has 0 tokens");
    assertEq(IERC1155(artAddress).balanceOf(user1, 1), 1, "user1 now has 1 token");
}
```

## Impact

- The entire pausing mechanism for `PhiNFT1155` is non-functional.
- In an emergency (exploit, bug, or governance decision), the owner cannot stop NFT transfers.
- This defeats the purpose of having a pause feature, which is a critical safety mechanism in NFT contracts.
- Medium severity because it is a broken core security feature, even if not immediately exploitable for direct theft.

## Recommendation

Inherit from `ERC1155PausableUpgradeable` instead of `PausableUpgradeable` and implement the required override for `_update`.

```diff
- import { PausableUpgradeable } from "@openzeppelin/contracts-upgradeable/utils/PausableUpgradeable.sol";
+ import { ERC1155PausableUpgradeable } from 
+     "@openzeppelin/contracts-upgradeable/token/ERC1155/extensions/ERC1155PausableUpgradeable.sol";

contract PhiNFT1155 is
    ...
    ReentrancyGuardUpgradeable,
-   PausableUpgradeable,
+   ERC1155PausableUpgradeable,
    ...
{
+   function _update(address from, address to, uint256[] memory ids, uint256[] memory values)
+       internal
+       override(ERC1155PausableUpgradeable, ERC1155SupplyUpgradeable)
+   {
+       super._update(from, to, ids, values);
+   }
}
```

This ensures that when the contract is paused, all ERC1155 operations (`safeTransferFrom`, `safeBatchTransferFrom`, minting via `claimFromFactory`, etc.) will revert.

---
