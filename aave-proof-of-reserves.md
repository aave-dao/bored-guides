# Proof of Reserve

**Author**: [BGD Labs](https://bgdlabs.com/)

---

> **Disclaimer**: This guide does NOT guarantee 100% correctness or completeness. Users MUST understand the context of application and have sufficient technical and product expertise. As a contributor and/or maintainer, always act defensively, whether trying to understand the system, testing new developments, or involving external security parties and procedures.

## 1. Introduction

The Aave Proof of Reserve system is an extra safeguard for pool reserves, monitoring collateralization data from [Chainlink Proof of Reserve feeds](https://chain.link/proof-of-reserve). If a covered asset's total supply exceeds the reserves reported by its feed, the system can quickly freeze the affected reserve, thereby protecting the remaining pool participants.

This guide covers the current status of the PoR system in the Aave Protocol and procedures for covering new assets.

## 2. System overview

### 2.1 Repository structure

The [repository](https://github.com/aave-dao/aave-proof-of-reserve/) contains all relevant contracts, tests and deployment scripts. For full mechanics and technical details on contracts and system design, see the [README](https://github.com/aave-dao/aave-proof-of-reserve/blob/main/README.md). Overall, the system has four main contracts:

- [`ProofOfReserveAggregator`](https://github.com/aave-dao/aave-proof-of-reserve/blob/main/src/contracts/ProofOfReserveAggregator.sol): Registry of assets and their Chainlink PoR feeds (and optional bridge wrappers).
- [`ProofOfReserveExecutor`](https://github.com/aave-dao/aave-proof-of-reserve/blob/main/src/contracts/ProofOfReserveExecutorV3.sol): Holds the list of assets monitored per pool and executes the emergency action (freeze + set LTV to zero).
- [`AvaxBridgeWrapper`](https://github.com/aave-dao/aave-proof-of-reserve/blob/main/src/contracts/AvaxBridgeWrapper.sol): Avalanche-specific helper that sums the total supply across the active and deprecated bridge contracts for a given asset.
- [`ProofOfReserveKeeper`](https://github.com/aave-dao/aave-proof-of-reserve/blob/main/src/contracts/ProofOfReserveKeeper.sol): Chainlink Automation-compatible contract that continuously monitors the system and triggers emergency action automatically when needed.

### 2.2 Current status

Currently, Proof of Reserve is active on the **Avalanche network only**, covering bridge assets (see [assets covered](https://github.com/aave-dao/aave-proof-of-reserve?tab=readme-ov-file#assets-covered)) whose backing is held on Ethereum and reported via Chainlink cross-chain PoR feeds. For current role assignments and contract ownership, see the [Aave Permissions Book](https://github.com/bgd-labs/aave-permissions-book).

## 3. Procedures

> [!NOTE]
> The PoR system is owned by the Aave Governance smart contracts, which means that adding or removing a reserve from the covered assets MUST be done through a relevant governance proposal (activating the PoR system on a new network, or simply enabling or disabling a new asset on Avalanche).

### 3.1 Adding a new asset to PoR monitoring

#### Step 1: Validate the PoR feed

1. Confirm a Chainlink Proof of Reserve feed exists and is live for the asset on the target network. If it does not exist, request it from Chainlink.
2. Verify the feed reports reserves in the same units as the asset's total supply, paying special attention to decimals. The Aggregator compares raw values directly; a mismatch will cause the asset to be incorrectly flagged as unbacked or never flagged at all.
3. If the asset has been bridged through multiple contracts (current and deprecated), confirm an `AvaxBridgeWrapper` is deployed that correctly aggregates supply across all of them.

#### Step 2: Enable in the Aggregator and Executor

Add the PoR feed to the proposal payload, and enable it via the relevant functions:

- `ProofOfReserveAggregator`: call `enableProofOfReserveFeed` (plain reserve) or `enableProofOfReserveFeedWithBridgeWrapper` (reserve with a bridge wrapper).
- `ProofOfReserveExecutor`: call `enableAssets` with the reserve as input.

### 3.2 Removing an asset from PoR monitoring

Prepare the proposal payload with the relevant calls:

- `ProofOfReserveAggregator`: call `disableProofOfReserveFeed` with the reserve as input.
- `ProofOfReserveExecutor`: call `disableAssets` with the reserve as input.

### 3.3 Automation

The `ProofOfReserveKeeper` is registered as a [Chainlink Automation](https://automation.chain.link/) upkeep. It runs off-chain on a recurring basis, and when an unbacked and unfrozen reserve is detected, the automation performs the emergency action.

The upkeep registration and funding are managed through the aave-robot infrastructure. Any changes to the keeper contract address or upkeep configuration require a governance proposal.

Note that the execution of the emergency action is permissionless; anyone can trigger it directly if the conditions are met, independent of the Keeper.

### 3.4 Monitoring triggers

To detect PoR emergency actions on-chain, off-chain monitors should observe the following events emitted by the `ProofOfReserveExecutor`:

- `EmergencyActionExecuted`: emitted when an emergency action is triggered on the pool.
- `AssetIsNotBacked`: emitted for each reserve found unbacked during that execution.

## 4. General tips

- The Proof of Reserve system is designed to accommodate different types of reserve backing - cross-chain bridge assets and off-chain real-world assets are both valid use cases.
- Independent of the types of reserve backing, the core invariant is always the same: the total supply of the monitored asset must not exceed the value reported by its Chainlink PoR feed.
- The Aggregator compares the asset's total supply directly against the latest answer from the PoR feed. If the asset and the feed have different decimal precisions, the comparison will be incorrect. The decimal units are crucial for the system to work as expected.
