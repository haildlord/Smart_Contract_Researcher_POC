# H-03 | Signature Replay in `createArt` Allows Impersonation of Artist and Theft of Royalties

**Author:** Hailthelord

**Protocol:** Phi

**Proof Link:** https://github.com/code-423n4/2024-08-phi-findings/issues/70

**Severity:** High  

**Category:** Signature  

**Type:** Signature Replay / Missing Domain Separation  

**Protocol:** Phi Protocol  

**Submitted via:** Code4rena  

## Summary

The `createArt()` function in `PhiFactory` accepts a signature that only covers `signCreateData` (art ID, expiry, cred data). It does **not** bind the signature to the `CreateConfig` struct (especially the `receiver` — royalty recipient — and `artCreator`). Combined with the fact that `createERC1155Internal` succeeds even when the contract already exists, this allows anyone to front-run a legitimate `createArt` transaction, reuse the signature with a malicious config, and steal all future royalties.

## Vulnerability Details

### Root Causes

1. **Signature does not include `CreateConfig`**  
   The signed message only contains `expiresIn + artIdURL + credData`. Critical economic parameters such as:
   - `receiver` (royalties recipient)
   - `artCreator`
   - `royaltyBPS` related settings
   are completely unsigned and can be supplied arbitrarily by an attacker.

2. **No binding to specific submitter**  
   There is no check that `msg.sender` matches the intended creator.

3. **Idempotent creation allows double success**  
   `createERC1155Internal` does not revert if the PhiNFT1155 contract for that `artId` already exists. Both the attacker’s front-run transaction and the original transaction succeed.

### Attack Flow

1. Legitimate user prepares a valid signature + `CreateConfig` and broadcasts `createArt(...)`.
2. Attacker sees the transaction in the mempool and front-runs it with the **same signature** but their own `CreateConfig` where `receiver = attacker`.
3. Attacker’s transaction succeeds → `ArtData.receiver` is set to the attacker.
4. Original user’s transaction also succeeds (because creation is idempotent) → events are emitted normally.
5. All future royalty payments and rewards routed through `PhiRewards` go to the attacker instead of the legitimate artist/recipient.
6. Additionally, the attacker (now recorded as `artCreator`) can call privileged functions like `updateRoyalties()` and `updateArtSettings()`.

## Impact

- **Direct theft of royalties** — Attacker receives all future royalty payments intended for the artist.
- **Privilege escalation** — Attacker gains the ability to call `PhiNFT1155::updateRoyalties()` and `PhiFactory::updateArtSettings()`, allowing them to change royalty rates, max supply, mint fees, etc.
- **Loss of funds for legitimate artists and creators**.
- High severity because the attack is cheap to execute (just front-running gas) and has permanent economic consequences.

## Proof of Concept (Full Code)

Drop this test into `PhiFactory.t.sol` and run with `forge test --match-test testKuprum_ImpersonateArtist`:

```solidity
function testKuprum_ImpersonateArtist() public {
    // The owner prepares the signature, the config,
    // and submits the `createArt` transaction
    string memory artIdURL = "sample-art-id";
    bytes memory credData = abi.encode(1, owner, "SIGNATURE", 31_337, bytes32(0));
    bytes memory signCreateData = abi.encode(expiresIn, artIdURL, credData);
    bytes32 createMsgHash = keccak256(signCreateData);
    bytes32 createDigest = ECDSA.toEthSignedMessageHash(createMsgHash);
    (uint8 cv, bytes32 cr, bytes32 cs) = vm.sign(claimSignerPrivateKey, createDigest);
    if (cv != 27) cs = cs | bytes32(uint256(1) << 255);
    IPhiFactory.CreateConfig memory config =
        IPhiFactory.CreateConfig(artCreator, receiver, END_TIME, START_TIME, MAX_SUPPLY, MINT_FEE, false);
    
    // user1 observes `createArt` transaction in the mempool, and frontruns it,
    // reusing the signature, but with their own config where user1 is the receiver
    vm.deal(user1, 1 ether);
    vm.startPrank(user1);
    IPhiFactory.CreateConfig memory user1Config =
        IPhiFactory.CreateConfig(artCreator, user1, END_TIME, START_TIME, MAX_SUPPLY, MINT_FEE, false);
    phiFactory.createArt{ value: NFT_ART_CREATE_FEE }(signCreateData, abi.encodePacked(cr, cs), user1Config);

    // Owner's `createArt` succeeds; there is also no difference in the `ArtContractCreated` event
    vm.startPrank(owner);
    phiFactory.createArt{ value: NFT_ART_CREATE_FEE }(signCreateData, abi.encodePacked(cr, cs), config);

    // Verify that user1 is now the royalties recepient
    uint256 artIdNum = 1;
    IPhiFactory.ArtData memory updatedArt = phiFactory.artData(artIdNum);
    assertEq(updatedArt.receiver, user1, "user1 should be the royalties recepient");
}
```

## Recommendation

Implement the following mitigations:

1. **Include `CreateConfig` in the signed message** (or at minimum the critical fields: `receiver`, `artCreator`, `maxSupply`, etc.).

2. **Bind the signature to a specific submitter** (include `msg.sender` or a dedicated `submitter` field in the signed data).

3. **Make `createERC1155Internal` revert if the contract already exists** for that `artId`, so only the first valid transaction succeeds.

Example direction for the fix in `createArt()`:

```solidity
function createArt(
    bytes memory signCreateData,
    bytes memory signature,
    CreateConfig memory config
) external payable {
    // 1. Recover signer from signature
    // 2. Verify that signCreateData now also commits to keccak256(abi.encode(config))
    // 3. Optionally verify msg.sender == intended submitter
    // 4. Proceed only if checks pass
}
```

Additionally, consider using EIP-712 typed data hashing with proper domain separation for better security and readability.

