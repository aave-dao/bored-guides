# Umbrella maintenance/expansion

**Author**: [BGD Labs](https://bgdlabs.com/)

---

> **Disclaimer**: This guide does NOT guarantee 100% correctness or completeness. Users MUST understand the context of application and have sufficient technical and product expertise. As a contributor and/or maintainer, always act defensively, both trying to understand the system, test new development, or even involving external security parties and procedures.
> 

## 1. Introduction

Umbrella is the Aave DAO's reserve-specific backstop for the deficit. Each covered reserve has its own staking pool, where users stake wrapped aTokens (or GHO), earn rewards, and bear the slashing risk. If bad debt occurs, staked funds are automatically slashed to cover the deficit without a governance vote.

For contract mechanics and implementation details, see the [repository documentation](https://github.com/aave-dao/aave-umbrella). For design rationale and activation precedent, see the [original Umbrella proposal](https://governance.aave.com/t/bgd-aave-safety-module-umbrella/18366) and the [first activation ARFC](https://governance.aave.com/t/arfc-aave-umbrella-activation/21521).

## 2. Add a new asset to Umbrella

The initial Ethereum Core activation covered GHO, USDC, USDT, and WETH. Any subsequent asset addition on an already active Umbrella instance follows this flow.

### 2.1 Asset eligibility

Considerations:

- The asset should be materially borrowed on the pool, with possible deficit risk. It is important to evaluate the pricing model of collateral backing the asset, as in cases where the feed is pegged to a static reference, deficit could potentially not materialise in the borrowed asset (e.g., wstETH if de-pegging on secondary market).
The initial activation of Umbrella on v3 Ethereum is a good reference for configuration of all previous aspects.
- There should be enough expected staking demand to sustain a dedicated Umbrella market.

### 2.2 Define parameters

Parameters should be defined by the Aave Finance Committee before the proposal is posted. The key parameters are: covered reserve, target liquidity, reward tokens and emissions (`maxEmissionPerSecond`, `distributionEnd`, `rewardPayer`), cooldown and unstake window, deficit offset, liquidation fee, and underlying oracle. Technical service providers are responsible for the oracle setup and payload implementation.

### 2.3 Confirm operational ownership

Before moving forward, the proposal must make clear:

- Who will manage post-launch reward updates (typically the Aave Finance Committee, but network foundations or other entities may co-fund or co-manage rewards).
- Where reward tokens come from and how the `rewardPayer` address gets funded.
- Whether any treasury bridging/top-ups are needed (especially on L2).
- Whether any committee or steward needs new permissions.

The ARFC should cover all of the above: the governance case for adding the asset, the full parameter package, reward funding plan, and operational ownership. Once the forum discussion stabilizes, it moves to Snapshot.
The initial activation of Umbrella on v3 Ethereum is a good reference for configuration of all previous aspects.

### 2.4 AIP preparation and execution

If Snapshot passes, the AIP implements the agreed package. At a high level, the governance payload (typically inheriting `UmbrellaExtendedPayload`) does the following atomically via `complexTokenCreations()`:

1. Creates the StakeToken (underlying waToken/GHO, name suffix, cooldown, unstake window).
2. Sets slashing configuration (links StakeToken to reserve, oracle, liquidation fee).
3. Initializes rewards in RewardsController (target liquidity, reward configs).
4. Sets deficit offset (must include any pre-existing pool deficit).

For payload structure details, see [`UmbrellaExtendedPayload`](https://github.com/aave-dao/aave-umbrella/blob/main/src/contracts/payloads/UmbrellaExtendedPayload.sol) and [`IUmbrellaEngineStructs.TokenSetup`](https://github.com/aave-dao/aave-umbrella/blob/main/src/contracts/payloads/IUmbrellaEngineStructs.sol).

### 2.5 Post-execution checks

After execution, verify that the StakeToken is linked to the correct reserve, slashing configuration is active, and rewards are live and funded. For ongoing maintenance procedures (reward management, deficit coverage, etc.), see section 4.

## 3. Expand Umbrella to a new network

Unlike adding a single asset, this requires deploying the full Umbrella contract set, assigning all roles, setting up reward funding on a new network, and activating an initial set of covered reserves.

### 3.1 Network readiness

Before discussing Umbrella itself, confirm:

- Aave governance can already execute on the target network.
- The pool is on Aave v3.3+.
- The DAO has treasury / operational rails to support rewards on that network.

### 3.2 Scope and operating model

The proposal should define:

- Which pool instance is in scope and which reserves are covered on day one. Evaluate pool size, borrow composition, and deficit relevance. The initial activation prioritized Ethereum, Arbitrum, Avalanche, and Base.
- Parameter package for each covered asset (same as 2.2).
- The Aave Finance Committee as the entity managing rewards and deficit offset coverage.
- Reward budget, duration, and how the `rewardPayer` will be funded on the target network.
- Whether a `DeficitOffsetClinicSteward` should be deployed to cover pre-existing pool deficit without individual governance proposals (see 4.3).

### 3.3 ARFC, Snapshot, and AIP

The ARFC for a network expansion should cover:

- Network rationale and strategic case.
- Initial reserve set with full parameter packages.
- Role assignments.
- Reward funding plan.
- Post-launch monitoring expectations.

After Snapshot passes, the Umbrella system contracts are deployed on the target network (see [deployment scripts](https://github.com/aave-dao/aave-umbrella/blob/main/scripts/deploy/DeployUmbrella.s.sol) and repository documentation for details). The governance payload then handles system initialization and initial asset coverage:

The payload connects Umbrella to the pool, assigns all operational roles, and activates initial asset coverage (same per-asset setup as in 2.4).

### 3.4 Post-execution checks

After execution, verify that Umbrella is connected to the pool, all roles are assigned, and the initial asset coverage is active. Ongoing maintenance procedures from section 4 apply.

Additionally, for new network deployments:

- Track early liquidity growth versus target - both under- and over-capitalization require attention.
- Evaluate whether more assets on that network should wait or follow quickly based on initial performance.

## 4. Ongoing operations

### 4.1 Reward management

The Aave Finance Committee (holding `REWARDS_ADMIN_ROLE` on RewardsController) manages rewards without governance votes:

- **Emission rate**: adjust `maxEmissionPerSecond` up or down in response to staking demand and APY targets.
- **Distribution period**: extend `distributionEnd` before it expires. Emissions stop automatically at the configured timestamp - if nobody extends it, rewards simply stop.
- **Disabling rewards**: set emissions to zero if a StakeToken is being wound down.

Reward funding requires periodic attention:

- The `rewardPayer` address must stay funded and have sufficient allowance to the RewardsController. If the local Collector lacks sufficient reward tokens, governance proposals are needed to bridge funds from Ethereum.

### 4.2 Slashing automation

Slashing is triggered by a `SlashingRobot` - an automation contract deployed per Umbrella instance. The robot periodically checks all covered reserves for slashable deficit and executes slashing when conditions are met. Individual reserves can be disabled on the robot by its guardian (owner). Each network expansion requires deploying and registering a robot for the new Umbrella instance.

### 4.3 Deficit offset management

When Umbrella is initialized for a reserve, the deficit offset is automatically set to the current pool deficit, preventing instant slashing of new stakers. Good practice is to set the offset higher than the current deficit during initialization, so that early stakers are not slashed by dust deficit accumulating before deposits come in.

**Increasing the offset** (`setDeficitOffset`) - governance only (`DEFAULT_ADMIN_ROLE`). Over time, the offset should be recalibrated to reflect the protocol's capacity to absorb losses - for example, based on accumulated liquidation revenue for that reserve (see the [Revenue-indexed deficit offsets ARFC](https://governance.aave.com/t/arfc-revenue-indexed-deficit-offsets-for-umbrella/24000) for precedent).

**Covering the offset** - two paths:

- Via `DeficitOffsetClinicSteward` (if deployed): the Aave Finance Committee can cover the offset from the Collector without a governance proposal. The steward requires `COVERAGE_MANAGER_ROLE` on Umbrella and sufficient allowance from the Collector.
- Via governance proposal: in practice, increasing the offset and covering existing deficit are usually done in the same proposal.

Covering the offset eliminates pre-existing deficit on the pool without slashing stakers.

### 4.4 Covering deficit on reserves without Umbrella

Reserves that are not covered by Umbrella can still accumulate deficit on the pool. To cover it, governance can use `coverReserveDeficit` on the Umbrella contract (`COVERAGE_MANAGER_ROLE` required). This was used for CRV and ENS in the [first deficit offset update proposal](https://governance.aave.com/t/arfc-revenue-indexed-deficit-offsets-for-umbrella/24000). The governance payload pulls aTokens from the Collector to cover the deficit. If the Collector does not hold enough aTokens of the required asset, the tokens must be acquired and supplied to Aave on behalf of the Collector before the proposal can execute.

### 4.5 Guardian actions

- **Pause/unpause**: the Protocol Guardian can pause individual StakeTokens. While paused, no deposits, withdrawals, or slashing can occur.
- **Rescue**: the Rescue Guardian can recover funds mistakenly sent to Umbrella contracts or StakeTokens.

## 5. General tips

- Each StakeToken covers one reserve on one pool on one network. There is no cross-network or cross-pool coverage aggregation.
- If bad debt exceeds the total staked amount in a StakeToken, only the available funds are slashed. The remaining deficit stays on the pool uncovered.
- Coverage after slashing is a separate step: slashed funds sit in the Collector until the coverage manager executes a transaction to eliminate the deficit on the pool.
- Users cannot directly stake rebasing aTokens - the system requires wrapped versions (stataTokens). The UI abstracts this, but external integrators must handle wrapping themselves.
- The emission curve has three sectors: boosted emissions below target liquidity (reaching max at target), linear decrease from 100% to 80% of max between target and 120% of target, and flat 80% above 120%. See [RewardsController README](https://github.com/aave-dao/aave-umbrella/blob/main/src/contracts/rewards/README.md) for details.
- Umbrella does not deploy on pools below meaningful size - the operational overhead (reward funding, monitoring, robot deployment) must be justified by the pool's deficit risk profile.