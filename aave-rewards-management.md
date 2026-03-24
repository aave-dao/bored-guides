# Aave rewards (RewardsController/Umbrella) configuration

**Author**: [BGD Labs](https://bgdlabs.com/)

---

> **Disclaimer**: This guide does NOT guarantee 100% correctness or completeness. Users MUST understand the context of application and have sufficient technical and product expertise. As a contributor and/or maintainer, always act defensively, whether trying to understand the system, testing new developments, or involving external security parties and procedures.
> 

## 1. Introduction

Aave uses two distinct on-chain reward mechanisms: the *RewardsController* for Aave V3 liquidity mining and the *Umbrella RewardsController* for staking rewards on Umbrella assets. This repository ([aave-rewards-configuration](https://github.com/aave-dao/aave-rewards-configuration)) provides tooling for managing both.

For **v3 liquidity mining**, new campaigns should be managed via Merkle distribution systems, such as Merit/MASiv, as the on-chain RewardsController adds unnecessary complexity and maintenance overhead. At the time of writing, the only active on-chain LM program is aUSDS on Aave V3 Ethereum Core, managed by ACI, and **our recommendation is to, going forward, not activate more on-chain LM.** The tooling in this repository remains available for maintaining existing programs.

**Umbrella rewards** are configured and updated on-chain via the Umbrella RewardsController, using the PermissionedPayloadsController timelock infrastructure.

## 2. Repository and tooling overview

This repository ([aave-rewards-configuration](https://github.com/aave-dao/aave-rewards-configuration)) contains CLI generators, test utilities, scripts, and documentation for configuring both Aave V3 liquidity mining and Umbrella rewards. It includes:

- A **CLI generator** that bootstraps the required Solidity test scripts for creating, updating, or modifying rewards configurations. Run `npm run generate --help` for all available commands. The generator supports three actions: setting up new liquidity mining, updating existing liquidity mining, and updating existing Umbrella rewards.
- **Test base classes** for liquidity mining setup ([`LMSetupBaseTest.sol`](https://github.com/aave-dao/aave-rewards-configuration/blob/main/tests/utils/LMSetupBaseTest.sol)) and updates ([`LMUpdateBaseTest.sol`](https://github.com/aave-dao/aave-rewards-configuration/blob/main/tests/utils/LMUpdateBaseTest.sol)), and for umbrella rewards ([`UmbrellaRewardsBaseTest.sol`](https://github.com/aave-dao/aave-rewards-configuration/blob/main/tests/utils/UmbrellaRewardsBaseTest.sol)) that validate configurations before activation of rewards.
- **Example tests** simulating umbrella reward updates, liquidity mining activation, and emission configuration changes and generating encoded calldata for Safe execution.
- A **Seatbelt report** integration via Tenderly for umbrella rewards validation.

For detailed setup instructions, see the [repository README](https://github.com/aave-dao/aave-rewards-configuration/blob/main/README.md).

### Using the CLI generator

The CLI generator can be used for both **liquidity mining** (setup and updates) and **Umbrella rewards** (updates). Run `npm run generate` and select the desired action:

1. Setup new liquidity mining
2. Updating existing liquidity mining
3. Updating existing Umbrella Rewards

The CLI walks through pool selection, asset selection, reward selection, and parameter configuration — displaying current on-chain values where applicable. It generates Solidity test scripts that validate the configuration and emit encoded calldata for Safe execution.

To emit calldata without sending transactions: `forge test --mp tests/<GeneratedTestFile>.t.sol -vv`

The exact command will be generated in the comments of the output test file.

## 3. Aave V3 liquidity mining (RewardsController)

Liquidity mining on Aave V3 is managed by the **Emission Admin** (the address with permission to configure reward emissions) via the Emission Manager contract. The Emission Admin role is assigned per reward token i.e either underlying/aToken/vToken, typically at the time of asset listing.
The Emission Admin can:

- **Create new LM programs**: Configure assets to receive rewards by calling `configureAssets()` on the Emission Manager.
- **Update emissions**: Modify `emissionPerSecond` via `setEmissionPerSecond()` and `distributionEnd` via `setDistributionEnd()`.
- **Stop programs**: Set emissions to zero or move the distribution end to halt rewards.

### Admin roles and Rewards Vault

At the time of writing, the Emission Admin for active LM programs is held by ACI via their [multi-sig](https://etherscan.io/address/0xac140648435d03f784879cd789130F22Ef588Fcd). This same address also serves as the **Rewards Vault** — the address holding the LM reward tokens. The Rewards Vault address is configured in the `PullRewardsTransferStrategy` contract; if it needs to change, a new Transfer Strategy must be deployed pointing to the updated vault.

The flow requires: funds in the Rewards Vault, an ERC-20 approval from the Rewards Vault to the Transfer Strategy contract (typically `PullRewardsTransferStrategy`), and the `configureAssets()` call with the appropriate parameters.

The CLI generator scaffolds all required test files. Running the generated tests via `forge test -vv` emits the selector-encoded calldata that can be submitted through Safe.

For full details on the mechanics, parameters, and examples, see [docs/AaveV3Incentives.md](https://github.com/aave-dao/aave-rewards-configuration/blob/main/docs/AaveV3Incentives.md).

## 4. Umbrella rewards (Umbrella RewardsController)

Umbrella rewards are configured on the Umbrella RewardsController and are managed via the **PermissionedPayloadsController** — a timelock-based execution system.

### How the timelock works

The `REWARDS_ADMIN` role on the Umbrella RewardsController has been assigned to the [PermissionedPayloadsController](https://etherscan.io/address/0xF86F77F7531B3374274E3f725E0A81D60bC4bB67). This means updates go through a controlled process:

1. The **PayloadsManager** ([ALC Multi-sig](https://etherscan.io/address/0x22740deBa78d5a0c24C58C740e3715ec29de1bFa)) calls `createPayload()` on the PermissionedPayloadsController with the calldata for `configureRewards()`.
2. The payload is automatically queued with a timelock (1 day at time of configuration).
3. During the timelock, the **Guardian** (currently BGD Labs) can review the payload and cancel it if misconfigured.
4. Once the timelock passes, the Aave Robot (or anyone) can execute the payload.

Both the PayloadsManager and the Guardian can cancel payloads. The PayloadsManager can only create and cancel — execution is permissionless after the timelock.

For more on the PermissionedPayloadsController, see the [overview documentation](https://github.com/bgd-labs/aave-governance-v3/blob/main/docs/permissioned-payloads-controller-overview.md).

### Updatable parameters

The following parameters can be updated for an Umbrella asset-reward pair via the PermissionedPayloadsController:

- `maxEmissionsPerSecond`: Maximum emission rate, reached when `targetLiquidity` is deposited.
- `distributionEnd`: Timestamp when emissions stop.
- `rewardPayer`: Address funding the rewards (typically the Aave Collector).

Note: `targetLiquidity` and other structural parameters can only be changed via Aave Governance.

Before updating, ensure the `rewardPayer` has sufficient funds and approval for the new configuration.

### Admin roles

At the time of writing, Umbrella rewards updates are managed by the [**Aave Finance Committee**](https://etherscan.io/address/0xF86F77F7531B3374274E3f725E0A81D60bC4bB67), which is the PayloadsManager of the [Umbrella PPC](https://etherscan.io/address/0xF86F77F7531B3374274E3f725E0A81D60bC4bB67). The AFC can submit reward configuration payloads through the PPC timelock.

Additionally, the **Guardian** role is given to a technical/security provider (e.g., BGD Labs), which can cancel any payload during the timelock period if a misconfiguration is detected.

For full details, see [docs/UmbrellaRewards.md](https://github.com/aave-dao/aave-rewards-configuration/blob/main/docs/UmbrellaRewards.md).

## 5. General tips

- **Prefer Merkle distribution for liquidity mining**: For any new LM campaign, use Merit/MASiv rather than the on-chain Rewards Controller.
- **Always run tests before deployment**: Particularly for liquidity mining, the index has known edge cases/limitations. All tests in `LMSetupBaseTest`,`LMUpdateBaseTest` must pass before proceeding.
- **Ensure sufficient funds and approvals**: The Rewards Vault (for LM) or `rewardPayer` (for Umbrella) must have adequate balances and approvals for the configured distribution period. It is advisable to fund at least 3 months ahead.
- **Reuse existing Transfer Strategies**: If a `PullRewardsTransferStrategy` is already deployed for a given network with the correct rewards vault, rewards admin, and incentives controller, reuse it.
- **Use the CLI generator**: The repository provides a generator (`npm run generate`) that scaffolds all necessary scripts and tests, reducing the chance of manual error.
- **Validate calldata via Seatbelt**: For Umbrella rewards, the repository supports Tenderly-based seatbelt reports to validate calldata before submission.