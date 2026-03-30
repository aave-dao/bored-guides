# Risk Agents maintenance/expansion

**Author**: [BGD Labs](https://bgdlabs.com/)

---

> **Disclaimer**: This guide does NOT guarantee 100% correctness or completeness. Users MUST understand the context of application and have sufficient technical and product expertise. As a contributor and/or maintainer, always act defensively, both trying to understand the system, test new development, or even involving external security parties and procedures.
> 

## 1. Introduction

In addition to risk parameter updates done via standard Aave governance proposals, the Aave DAO operates a complementary risk parameter update system built on the [Chaos Agents](https://github.com/ChaosLabsInc/chaos-agents) middleware. This system sits between the Chaos Risk Oracle (an off-chain risk engine operated by Chaos Labs) and the Aave protocol, allowing certain pre-approved risk parameters to be updated in near-real-time without requiring individual governance proposals for each change.

The system operates through three components:

- **Chaos Risk Oracle**: Chaos Labs' off-chain risk engine continuously monitors risk indicators (cap utilization, market volatility, interest rate conditions, etc.) and publishes optimised parameter recommendations on-chain. To simplify description, we will use Chaos Labs in this document as the risk oracle provider, but there is no technical limitation to have multiple/different parties assuming this role.
- **Chaos Agents middleware** ([AgentHub](https://github.com/ChaosLabsInc/chaos-agents/blob/main/src/contracts/AgentHub.sol)): An orchestrator contract that pulls oracle payloads, runs generic validation (circuit breakers, expiration checks, duplicate detection, minimum delay enforcement), and forwards updates to the appropriate agent contract.
- **Aave Risk Agents** ([aave-risk-agents](https://github.com/aave-dao/aave-risk-agents)): Lightweight, Aave-specific contracts that validate payloads against protocol rules and inject the updates into the Aave protocol.

Each agent corresponds to a single `updateType` from the Risk Oracle and covers multiple markets. The `AgentHub` exposes two main methods: `check()` (used by automation to determine if an update is ready) and `execute()` (to inject the update into the protocol).

### 1.1 Currently active agents

The following agents are currently active on Aave, migrated from the previous AGRS (Aave Generalised Risk Stewards) system via [chaos-agents-migration](https://github.com/bgd-labs/chaos-agents-migration):

| Agent | Purpose | Contract |
| --- | --- | --- |
| **Supply Cap Agent** | Automated supply cap updates | [AaveSupplyCapAgent.sol](https://github.com/aave-dao/aave-risk-agents/blob/main/src/contracts/agent/AaveSupplyCapAgent.sol) |
| **Borrow Cap Agent** | Automated borrow cap updates | [AaveBorrowCapAgent.sol](https://github.com/aave-dao/aave-risk-agents/blob/main/src/contracts/agent/AaveBorrowCapAgent.sol) |
| **Rates Agent** | Interest rate strategy updates (base rate, slope1, slope2, optimal point) | [AaveRatesAgent.sol](https://github.com/aave-dao/aave-risk-agents/blob/main/src/contracts/agent/AaveRatesAgent.sol) |
| **CAPO Agent** | CAPO price feed parameter updates (snapshotRatio, maxGrowthPercent) | [AaveCapoAgent.sol](https://github.com/aave-dao/aave-risk-agents/blob/main/src/contracts/agent/AaveCapoAgent.sol) |
| **Discount Rate Agent** | Pendle PT discount-rate feed updates | [AaveDiscountRateAgent.sol](https://github.com/aave-dao/aave-risk-agents/blob/main/src/contracts/agent/AaveDiscountRateAgent.sol) |
| **E-Mode Agent** | E-Mode category parameter updates (LTV, LT, LB) | [AaveEModeAgent.sol](https://github.com/aave-dao/aave-risk-agents/blob/main/src/contracts/agent/AaveEModeAgent.sol) |

### 1.2 Key contracts and references

- Chaos Agents core contracts: [ChaosLabsInc/chaos-agents](https://github.com/ChaosLabsInc/chaos-agents)
- Aave-specific agent contracts: [aave-dao/aave-risk-agents](https://github.com/aave-dao/aave-risk-agents)
- Chaos Agents docs: [Configuration Guide](https://github.com/ChaosLabsInc/chaos-agents/blob/main/docs/ConfigurationGuide.md), [Threat Modeling](https://github.com/ChaosLabsInc/chaos-agents/blob/main/docs/ThreatModeling.md)
- Migration reference: [bgd-labs/chaos-agents-migration](https://github.com/bgd-labs/chaos-agents-migration)
- Deployed addresses: available in the [aave-address-book](https://github.com/aave-dao/aave-address-book) under `AGENT_HUB`, `AGENT_HUB_AUTOMATION`, and `RANGE_VALIDATION_MODULE` entries per network.

---

## 2. Maintenance of existing risk agents

### 2.1 Whitelisting new markets for an existing agent

When a new reserve or e-mode category is listed on Aave and should be covered by automated risk updates, the corresponding agent must be configured to allow that market. This is done via a governance proposal whose payload calls:

```solidity
AGENT_HUB.addAllowedMarket(agentId, market);
```

Where `market` depends on the agent type:

- For **caps agents**, **rates agent**, **CAPO agent**, and **discount rate agent**: `market` is the **reserve address**.
- For **e-mode agent**: `market` is the **eModeId** (cast to `address`).

**Important considerations:**

- Verify you are passing the correct `agentId` for the target agent. To double-check, call `AGENT_HUB.getUpdateType(agentId)` and confirm the returned `updateType` matches the agent you intend to configure.
- The governance proposal that whitelists the market **must include an e2e injection fork test** to validate that the agent can successfully inject a risk update for the newly whitelisted market. This catches misconfiguration early (wrong agentId, incorrect market address, missing range configs, etc.).
- For Pendle PT e-modes or new PT discount-rate feeds, ensure that the corresponding price feed adapter is already deployed and configured on the Aave Oracle before whitelisting the market on the agent.

### 2.2 Monitoring automated risk updates

The technical service provider should continuously monitor the health of the automated risk update pipeline. Key monitoring points:

**Tracking injected updates:**

Monitor the `UpdateInjected` event on the `AgentHub` contract. For each event, validate that the injected parameters are correct with no second order consequences.

**Automation health:**

The injection of updates from the Risk Oracle into the protocol is performed by Aave Robots:

- On networks where **Chainlink Automation** is available, a dedicated `AgentHubAutomation` contract is registered with the [AaveCLRobotOperator](https://search.onaave.com/?q=ROBOT%20OPERATOR). The robot calls `check()` to detect pending updates and `execute()` to inject them.
- On networks where Chainlink Automation is **not available**, an alternative automation platform (e.g., Chainlink CRE) is used.

The technical team must monitor that the automation robots have **sufficient funds** (ex. LINK tokens for Chainlink Automation) to continue operating. If funding runs low, an action is needed to top up the robot.

**Handling expired or blocked updates:**

If updates from the Risk Oracle expire before the automation can inject them (e.g., due to insufficient robot funds or any other reason), the system will appear stalled. To unblock the pending updates contact Chaos Labs to push fresh updates if the stale ones are no longer valid.

### 2.3 Range configuration updates

The `RangeValidationModule` enforces per-agent bounds on how much a risk parameter can change in a single update. Over time, these ranges may need adjustment as market conditions evolve or new assets with different risk profiles are listed.

- **Default range** (applies to all markets unless overridden): update via `RANGE_VALIDATION_MODULE.setDefaultRangeConfig()`.
- **Per-market overrides**: update via `RANGE_VALIDATION_MODULE.setRangeConfigByMarket()`.

Range configuration changes can be performed by the AgentHub owner or the agent admin. When adjusting ranges, consider the `maxChange` value and whether `isRelativeChange` is set (percentage-based vs. absolute change limits).

### 2.4 Emergency procedures

- **Disabling an agent**: The agent admin (governance) can disable a specific agent via `AGENT_HUB.setAgentEnabled(agentId, false)`. This immediately stops all automated updates for that agent without affecting other agents on the same hub.
- **Revoking permissions**: In an emergency, governance can revoke the `RISK_ADMIN` role from the agent contract via `ACL_MANAGER.removeRiskAdmin(agentAddress)`, cutting off the agent's ability to modify protocol parameters entirely.

---

## 3. Expansion: adding new risk agents

Adding a new risk agent involves coordination between the risk service provider (Chaos Labs) and the technical service provider, followed by a governance proposal. This section is split into an operations guide covering the coordination workflow, and a technical guide covering implementation details.

### 3.1 Understanding sub-updateTypes

Risk agents sometimes batch multiple related risk parameters into a single update. For example, rate updates on Aave batch `baseRate`, `slope1`, `slope2`, and `uOpt` together. These sub-updateTypes do not exist on the AgentHub or Risk Oracle directly — they are specific to the protocol agent contract and should be handled carefully.

The risk team should discuss with the technical team whether to batch sub-updateTypes under a single agent or use separate agents for each. As a general rule:

- **Correlated risk params**: Batch together under one agent and `updateType` to ensure they are updated atomically.
- **Uncorrelated risk params**: Use separate agents with distinct `updateTypes`.

If a risk update has sub-updateTypes, the `expirationPeriod` and `minimumDelay` are shared across all sub-updateTypes for that agent. However, `maxChangeAllowed` and `isChangeRelative` on the `RangeValidationModule` can be configured differently per sub-updateType (e.g., different range bounds for `slope1` vs. `baseRate`).

### 3.2 Operations guide (risk ↔ tech coordination)

**Step 1: Scope and agree on the new agent**

The risk service provider identifies a new risk parameter that should be automated (e.g., a new parameter type, or extending an existing type to new networks). The risk and technical service providers agree on:

- The `updateType` string on the Risk Oracle.
- Whether sub-updateTypes should be batched under a single agent or split across separate agents.
- The validation rules (range configs, minimum delays, expiration periods).
- The target networks and pool instances.

**Step 2: Encoding format agreement**

Risk updates on the Risk Oracle are bytes-encoded. The technical team must share with the risk team the exact format or struct in which the agent contract expects to consume the risk update, so the risk team can prepare their off-chain infrastructure accordingly.

**Step 3: Agent contract preparation and constraint sharing**

Once the risk team shares the details about the risk update, the technical team should begin development of the protocol-specific agent contract (if one does not already exist). The technical team should share a high-level overview of the agent with the risk team, covering all on-chain constraints at every level — both constraints enforced by the protocol itself and additional validation enforced by the agent contract. For example, for cap updates on Aave, the cap value can never be set to 0.

All protocol-level constraints must be clearly communicated to the risk team so they can account for them in their off-chain model.

**Step 4: Risk Oracle preparation (Chaos Labs)**

Chaos Labs configures their off-chain risk engine and publishes the new `updateType` on the Risk Oracle. They must push **at least one test update** to the Risk Oracle at least **10 days before** the planned activation. This test update is used by the technical service provider to validate the end-to-end flow during payload testing.

**Step 5: Security validation and deployment (technical service provider)**

The technical service provider deploys the agent contract (if new), prepares the activation payload, and runs full security validation (see Section 3.4).

**Step 6: Governance proposal**

The activation is bundled into an AIP. The proposal should clearly state: which risk parameter is being automated, on which networks/instances, with what range constraints, and reference the test updates on the Risk Oracle.

### 3.3 Technical guide: preparing and deploying a new agent

### 3.3.1 Agent contract development

First, check if a suitable agent contract already exists in [aave-risk-agents](https://github.com/bgd-labs/aave-risk-agents). If so, skip to 3.3.2.

If a new agent contract is needed, create one following the existing patterns:

- [AaveBorrowCapAgent](https://github.com/aave-dao/aave-risk-agents/blob/main/src/contracts/agent/AaveBorrowCapAgent.sol)
- [AaveCapoAgent](https://github.com/aave-dao/aave-risk-agents/blob/main/src/contracts/agent/AaveCapoAgent.sol)
- [AaveDiscountRateAgent](https://github.com/aave-dao/aave-risk-agents/blob/main/src/contracts/agent/AaveDiscountRateAgent.sol)
- [AaveRatesAgent](https://github.com/aave-dao/aave-risk-agents/blob/main/src/contracts/agent/AaveRatesAgent.sol)
- [AaveEModeAgent](https://github.com/aave-dao/aave-risk-agents/blob/main/src/contracts/agent/AaveEModeAgent.sol)

Each agent implements `validate()` (Aave-specific validation logic) and `inject()` (protocol state change), and optionally `getMarkets()` for dynamic market lists.

### 3.3.2 Deploy scripts

Add deploy scripts in [aave-risk-agents/scripts](https://github.com/aave-dao/aave-risk-agents/tree/main/scripts) for each target network. Pay attention to:

- Passing the correct constructor params from the aave-address-book.
- The `updateType` suffix convention: use an empty suffix for non-Ethereum networks, and `_Core` or `_Prime` for Ethereum instances.

### 3.3.3 Deploying AgentHub and RangeValidationModule (if needed)

If the target network does not already have an `AgentHub` or `RangeValidationModule` in the aave-address-book, deploy them using the scripts in [ChaosLabsInc/chaos-agents/scripts](https://github.com/ChaosLabsInc/chaos-agents/tree/main/scripts). Deploy both `AgentHub` and `AgentHubAutomation` contracts, then add the deployed addresses to the aave-address-book.

### 3.3.4 Activation payload

The governance payload must perform the following actions **in this order**:

1. **Register the agent on the AgentHub**: Call `AGENT_HUB.registerAgent()`. This must happen first, as subsequent steps depend on the returned `agentId`.
2. **Configure range configs on the RangeValidationModule**: Set `setDefaultRangeConfig()` for the agent, and `setRangeConfigByMarket()` for any reserves that need different bounds. The module prioritizes per-market configs and falls back to the default config when none exists for a given reserve.
3. **Grant RISK_ADMIN role to the agent contract**: Call `ACL_MANAGER.addRiskAdmin(agentAddress)` to authorize the agent to modify protocol parameters.
4. **Register automation**:
    - **If Chainlink Automation exists on the network**: Call `register()` on the [AaveCLRobotOperator](https://search.onaave.com/?q=ROBOT%20OPERATOR). Use the `AGENT_HUB_AUTOMATION` contract (not `AGENT_HUB`) and pass the `agentId` in the `checkData` parameter. Fund the registration with LINK tokens from the Collector.
    - **If Chainlink Automation does not exist**: Register automation via an alternative platform (e.g., Chainlink CRE) **before** the proposal is created. Ensure the correct `agentId`s are passed in the `checkData`.

**Reference payloads:**

- [BaseActivateRiskAgentPayload.sol](https://github.com/aave-dao/aave-proposals-v3/blob/cb882c55b7b7576abb980dd3f664dd6420ca770d/src/20260130_Multi_ActivateCapoRiskAgentAndExpandRatesAgentOnMoreNetworks/BaseActivateRiskAgentPayload.sol)
- [BaseMigrationPayload.sol](https://github.com/bgd-labs/chaos-agents-migration/blob/d36ccff63cd780c4d47e710935ed1a547f54f221/src/contracts/BaseMigrationPayload.sol)

### 3.4 Security procedures: validating the activation payload

### 3.4.1 Validating test updates on the Risk Oracle

Before the activation payload is executed, confirm the Risk Oracle is correctly configured:

1. Confirm the `updateType` for the risk param exists on the Risk Oracle.
2. Verify that at least one test update has been pushed and is recent (pushed **at least 10 days before** activation). If stale, contact Chaos Labs to push a fresh update.
3. Confirm the test risk update will not interfere with the activation — sufficient time (`expirationPeriod`) must have passed by the time of activation.
4. For each test update, validate the following fields: `newValue` (correctly bytes-encoded), `market`, and `updateType` (on Ethereum, verify the `_Core` / `_Prime` suffix).
5. Verify that `newValue` is sane, correct for all parameters, and respects the configured range configs.
6. Perform an e2e injection test using the test update on the proposal activation (with snapshot diffs) to check for second-order effects on risk params. You can adjust the test block so the update is not expired at the time of testing.

### 3.4.2 Code diffs for new contracts

If any new contracts were deployed (AgentHub, RangeValidationModule, or agent contracts), perform code diffs:

- Compare against existing deployed contracts on other networks, **or**
- Compare against the source on GitHub: [chaos-agents contracts](https://github.com/ChaosLabsInc/chaos-agents/tree/main/src/contracts) and [aave-risk-agents contracts](https://github.com/aave-dao/aave-risk-agents/tree/main/src/contracts/agent).

There must be **zero or minimal code diffs** between the newly deployed contracts and the reference source.

### 3.4.3 Fork testing the full activation payload

Run e2e tests (including risk update injection) on a Foundry fork for **all markets and risk params**. Push mock updates on the Risk Oracle consistent with the test updates from 3.3.1.

Validate each of the following on the fork:

- **Agent registration**: Agents are registered on the AgentHub with correct and sane params.
- **Range configuration**: Default and per-market range configs on the `RangeValidationModule` are correct, including `maxChange` and `isRelativeChange`.
- **Agent contract immutables**: The agent contract points to the correct `AgentHub`; the `AgentHubAutomation` and `RangeValidationModule` point to the same correct `AgentHub`.
- **RISK_ADMIN role**: The agent contract has been granted the `RISK_ADMIN` role.
- **Automation registration** (if Chainlink): Automation is registered via `AaveCLRobotOperator` and funded with sufficient LINK tokens from the Collector.
- **Update types**: All agent `updateTypes` exist on the Risk Oracle.
- **Proxy admin** (if new AgentHub deployed): The `proxyAdmin` is set correctly and points to the Aave Governance `ExecutorLvl1`.

**Reference fork test:** [BaseMigrationPayload.t.sol](https://github.com/bgd-labs/chaos-agents-migration/blob/d36ccff63cd780c4d47e710935ed1a547f54f221/tests/BaseMigrationPayload.t.sol)

---

## 4. General tips

- Each agent on the AgentHub corresponds to exactly one `updateType` and multiple markets. If you need to automate a new parameter type, register a new agent — do not overload existing ones.
- On Ethereum, always verify the `updateType` suffix (`_Core` / `_Prime`) matches the target pool instance.
- The `RangeValidationModule` is an independent, immutable contract shared across agents on the same hub. Range configs are set per `(agentId, market)` pair.
- Agent contracts are designed to be lightweight and are expected to be redeployed when protocol upgrades change the injection interface. When redeploying, use `AGENT_HUB.setAgentAddress()` to update the hub's pointer and revoke/grant `RISK_ADMIN` accordingly.
- The `RISK_ADMIN` role MUST only be granted to the **agent contract** (`agentAddress`), never to the AgentHub, the AgentHubAutomation, or any other contract. The agent contract is the only entity that calls into the Aave protocol to modify parameters; granting the role elsewhere would bypass the agent-level validation logic.
- All privileged roles on the AgentHub (hub owner, agent admin, proxy admin) are held by the Aave Governance `ExecutorLvl1`. This means any configuration change — registering agents, updating range configs, enabling/disabling agents, upgrading the hub implementation — requires an Aave governance proposal. There is no admin multisig or off-chain key that can modify the system outside of governance.
- If the AgentHub needs to be upgraded (e.g., new hub version with no breaking changes), deploy the new implementation and use the ProxyAdmin to point the hub proxy to it. No agent re-registration is needed if the storage layout is preserved.