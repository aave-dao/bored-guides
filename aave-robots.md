# Aave Robots

**Author**: [BGD Labs](https://bgdlabs.com/)

---

> **Disclaimer**: This guide does NOT guarantee 100% correctness or completeness. Users MUST understand the context of application and have sufficient technical and product expertise. As a contributor and/or maintainer, always act defensively, whether trying to understand the system, testing new developments, or involving external security parties and procedures.
> 

## 1. Introduction

Aave Robots are the automation infrastructure powering permissionless actions across the Aave ecosystem. Robots trigger on-chain function calls that anyone is entitled to execute — the automation layer simply ensures these calls happen reliably and promptly without human intervention.

Automations are used across many parts of the DAO, including Governance v3, Proof of Reserve, Risk Agents, Umbrella, GHO GSM, and more.

**A list of all active robots per network can be found in the [aave-robots-list](https://github.com/aave-dao/aave-robots-list) repository.**

This guide covers the current automation infrastructure, access control, procedures for adding and managing robots, emergency controls, and monitoring responsibilities.

## 2. Automation infrastructure

All Aave DAO automations currently run on two Chainlink-based systems: **Chainlink Automation** and **Chainlink CRE (Chainlink Runtime Environment)**. Chainlink CRE, Initially will be activated on networks where Chainlink Automation is not available.

> **Note**: Previously, [Gelato Automation](https://www.gelato.network/) was used on networks without Chainlink Automation support. With Gelato deprecating their automation / web3 functions service on 31st March 2025, all automations have been migrated to Chainlink CRE.
> 

### 2.1 Repository

The primary repository for all robot infrastructure is [aave-governance-v3-robot](https://github.com/aave-dao/aave-governance-v3-robot). It contains the smart contracts (keeper contracts, robot operator), and the Chainlink CRE workflow code.

### 2.2 Chainlink Automation

[Chainlink Automation](https://automation.chain.link/) is the primary automation backend. It uses an on-chain keeper pattern: robot keeper contracts implement `checkUpkeep()` (off-chain simulation to detect if work is needed) and `performUpkeep()` (on-chain execution of the action). Chainlink's decentralized network of nodes monitors and executes these upkeeps.

On networks with Chainlink Automation, the [AaveCLRobotOperator](https://github.com/aave-dao/aave-governance-v3-robot/blob/main/src/contracts/AaveCLRobotOperator.sol) contract is used to manage upkeep registrations, cancellations, and administrative operations. The Robot Operator addresses for each network can be found on the [Aave Address Book](https://search.onaave.com/?q=ROBOT%20OPERATOR).

**Networks using Chainlink Automation**: Ethereum, Polygon, Avalanche, Arbitrum, Optimism, BNB Chain, and Base.

### 2.3 Chainlink CRE

[Chainlink CRE](https://docs.chain.link/) is used on networks where Chainlink Automation is not available. The CRE workflow runs off-chain on a cron schedule, calling `checkUpkeep()` on each configured robot contract. When work is needed, it signs and submits a CRE report to the [`MailboxCRE`](https://github.com/aave-dao/aave-governance-v3-robot/blob/main/src/contracts/MailboxCRE.sol) contract deployed on that network. The `MailboxCRE` then decodes the report and forwards the `performUpkeep()` call to the target robot contract.

The CRE workflow code and configuration live under [cre/](https://github.com/aave-dao/aave-governance-v3-robot/tree/main/cre) in the robot repository.

*No caller or forwarder checks are enforced on `MailboxCRE` — this is intentional, since the underlying robot actions are permissionless (anyone can call `performUpkeep` directly).*

**Networks currently using Chainlink CRE**: Gnosis, Ink, Linea, Plasma, Sonic, Celo, Scroll, XLayer, Mantle, MegaETH, and ZKsync.

## 3. Access control

### 3.1 Chainlink Automation

Access control for Chainlink Automation is **fully governed by Aave Governance**. All administrative actions — registering new upkeeps, cancelling existing ones, refunding, or withdrawing LINK — are performed via governance proposals that call the `AaveCLRobotOperator` contract. No off-chain entity can unilaterally add, remove, or reconfigure automations.

The `AaveCLRobotOperator` contract is owned by the relevant Aave Governance executor on each network.

### 3.2 Chainlink CRE

For Chainlink CRE, an **Aave DAO organization account** has been created on the Chainlink CRE platform. Email access to this account is shared with the elected service providers, who can add, update, or remove automations through the CRE platform's workflow management interface. While this is not directly on-chain governance-controlled like Chainlink Automation, the operational access is limited to the DAO's designated service providers.

## 4. Procedures

### 4.1 Registering a new automation (Chainlink Automation)

Registering a new automation on a network with Chainlink Automation requires a governance proposal. The proposal payload should call the `AaveCLRobotOperator` contract's `register()` function with the keeper contract address, upkeep name, gas limit, and initial LINK funding amount.

For the full contract interface, see [`AaveCLRobotOperator.sol`](https://github.com/aave-dao/aave-governance-v3-robot/blob/main/src/contracts/AaveCLRobotOperator.sol).

### 4.2 Cancelling an automation (Chainlink Automation)

Similarly, cancelling an existing automation requires a governance proposal calling the `cancel()` function on the `AaveCLRobotOperator`, passing the upkeep ID.

### 4.3 Adding a new automation (Chainlink CRE)

To add a new robot on a CRE-enabled network:

1. Add an entry to the `automations` array in `config.production.json`:

```json
{
  "address": "0x<robot-address>",
  "checkData": "0x",
  "automationContractType": "chainlink"
}
```

1. Ensure a `MailboxCRE` contract is deployed on that chain and its address is set as `mailboxAddress` in the config.
2. Simulate the workflow to verify correctness:

```bash
cre workflow simulate ./automation --target=production-settings
```

1. Deploy the updated workflow:

```bash
cre workflow deploy ./automation --target=production-settings
```

The `automationContractType` field accounts for interface differences between keeper contracts. Chainlink-style keepers have a two-step pattern: `checkUpkeep()` returns `performData`, and the workflow then encodes a separate `performUpkeep(performData)` call. Gelato-style keepers instead return the full encoded calldata directly from `checkUpkeep()`, ready to be forwarded as-is. Since the migration from Gelato to CRE did not involve redeploying existing keeper contracts, some networks still have Gelato-style contracts — this field tells the CRE workflow which encoding path to use for each robot.

For full setup and deployment instructions, see the [CRE automation README](https://github.com/aave-dao/aave-governance-v3-robot/blob/main/cre/automation/README.md).

### 4.4 Funding robot balances

### Chainlink Automation

Anyone with LINK tokens can permissionlessly fund a robot's upkeep by calling [`refillKeeper()`](https://github.com/aave-dao/aave-governance-v3-robot/blob/main/src/contracts/AaveCLRobotOperator.sol#L110) on the `AaveCLRobotOperator` contract, passing the upkeep ID and the LINK amount. No governance proposal is needed for top-ups.

### Chainlink CRE

CRE automations are funded by sending LINK or USDC on Ethereum mainnet to the Aave DAO CRE funding address:`0x434ae49ECaC5b122AA4e94586386929c7b1125a6`

Funding currently is managed by the Aave Finance Committee (AFC).

### 4.5 Emergency: disabling an automation

The automation contracts sitting on top of the permissionless methods include a mechanism to disable specific automations. For example, on the governance automation contract on Ethereum ([0xBa37...E257](https://etherscan.io/address/0xBa37F9eDC52f57caFA3a13ddfD655797Cc4FE257)), calling `toggleDisableAutomationById()` will cause the automation infrastructure to skip that automation. This is important for two scenarios:

- **Emergencies**: If protocol conditions require halting the automation immediately.
- **Ad-hoc execution**: If an automation action should be triggered manually rather than by the robot (e.g., for specific timing requirements).

This method can be called by the **owner** of the contract. Currently, this role is held by BGD Labs, but the plan is to migrate it to TokenLogic so that pausing automations can be handled by the elected service provider.

> **Note**: Since all automated actions are permissionless, disabling the robot does not prevent the action from being executed — it only prevents the automation infrastructure from triggering it. Anyone can still call the underlying function directly.
> 

## 5. Monitoring

The elected service provider responsible for robot maintenance must actively monitor the following:

### 5.1 Automation Balance

Each Chainlink Automation upkeep requires a LINK token balance to pay for execution. If the balance drops too low, the upkeep will stop executing.

To check whether a robot's balance is healthy, query the Chainlink Automation Registry on the relevant network:

1. Call `getMinBalance(upkeepId)` on the chainlink automation registry to get the minimum LINK balance required for the upkeep to remain active.
2. Call `getUpkeep(upkeepId)` on the registry to get the current balance of the upkeep.
3. Compare the current balance against the minimum balance — if it is approaching or below the minimum, a top-up is needed (see section 4.4).

The service provider should set up automated monitoring to detect low balances before they cause upkeep failures.

Similarly, automations running on Chainlink CRE require sufficient LINK / USDC balance in the DAO CRE address: [`0x434ae49ECaC5b122AA4e94586386929c7b1125a6`](https://etherscan.io/address/0x434ae49ECaC5b122AA4e94586386929c7b1125a6) to cover execution costs. The service provider must monitor this balance and top it up as needed through the Aave DAO CRE organization account.

### 5.3 Gas limit (Chainlink Automation)

> **Important**: Chainlink Automation enforces an **on-chain gas limit of 5 million gas** per upkeep execution. Any automation action that consumes more than 5M gas will fail and must be triggered manually.
> 

The service provider must monitor for cases where automated actions exceed this limit. A known scenario is governance payload execution: some governance proposals contain payloads whose execution gas cost exceeds 5M gas. When this occurs, the service provider must detect the failure and trigger the execution manually (by calling the permissionless function directly).

### 5.4 Robot health and execution

Beyond balance and gas monitoring, the service provider should track that automations are executing as expected — i.e., that governance proposals move through their lifecycle, that Proof of Reserve checks run, and that Umbrella slashing triggers correctly when conditions are met. Any unexpected failures or delays should be investigated promptly.

## 6. General tips

- All actions triggered by robots are permissionless. The robot infrastructure is a convenience and reliability layer, not a gatekeeper. If robots are down, any participant can execute the same actions on-chain.
- When deploying a new keeper contract, always verify that `checkUpkeep()` returns correct data and that `performUpkeep()` executes the intended action in a test environment before registering the automation.
- For Chainlink Automation, always account for the 5M gas limit when designing new keeper contracts. If an action may exceed this limit under certain conditions, plan for manual execution fallback.
- For Chainlink CRE, the cron schedule is configured in the workflow's `config.production.json` (currently `/5 * * * *`, i.e., every 5 minutes). Adjust this based on the time-sensitivity of the automation.
- All active robots can be found in the [aave-robots-list](https://github.com/aave-dao/aave-robots-list) repository. Whenever a robot is added or cancelled, the entries there must be updated to reflect the current state. Check the existing entries for the expected format. If the robot type is new, add a description in [`ROBOT_DESCRIPTIONS.md`](https://github.com/aave-dao/aave-robots-list/blob/main/ROBOT_DESCRIPTIONS.md) as well.