# Aave v3 asset listings procedure

**Author**: [BGD Labs](https://bgdlabs.com/)

---

> **Disclaimer**: This guide does NOT guarantee 100% correctness or completeness. Users MUST understand the context of application and have sufficient technical and product expertise. As a contributor and/or maintainer, always act defensively, whether trying to understand the system, testing new developments, or involving external security parties and procedures.
> 

## 1. Introduction

Listing a new asset on Aave v3 is a multi-stakeholder process involving risk analysis, technical/security due diligence, oracle configuration, and on-chain proposal execution. While the governance flow is well-documented in terms of TEMP CHECK > ARFC > AIP stages, this guide focuses on the operational and technical procedures around asset listings.

It is intended for:

- **Service providers creating listing proposals**
- **Risk service providers** performing risk analysis
- **Technical service providers** performing security/technical analysis
- **Governance delegates** reviewing listing proposals
- **External teams** seeking to list their assets on Aave

---

## 2. High-level listing flow

The lifecycle of an asset listing on Aave v3 follows these stages:

1. **Asset class check (AAcA)** - confirm the asset belongs to a pre-approved asset class.
2. **TEMP CHECK** - community appetite signal via forum discussion and Snapshot vote.
3. **ARFC** - triggers service provider work streams: risk analysis, oracle/pricing decision, and technical/security analysis.
4. **ARFC updated with final parameters** - risk service providers publish final listing parameters.
5. **Ad-hoc infrastructure deployment** - if needed: custom price feed, CAPO adapter, or other technical infrastructure deployed by the tech team.
6. **AIP** - on-chain proposal created by the service provider responsible for proposal creation, using the Config Engine, including initial seeding, reviewed via Seatbelt.
7. **On-chain vote** - AAVE holders vote.
8. **Execution** - proposal executed, asset live.

The following roles are involved in the process:

| Role | Responsibility in listing flow |
| --- | --- |
| **Proposal creator SP** | Initiates TEMP CHECK, manages ARFC process, creates AIP proposal code, coordinates with asset teams |
| **Risk SPs** | Risk analysis, define all listing parameters, oracle selection input, ongoing monitoring post-listing |
| **Tech SP** | Technical/security analysis, custom price feed deployment (if needed), Seatbelt review, proposal code review |
| **Asset team** | Provides documentation, answers questions during analysis, may provide seed tokens, implements recommendations (e.g., adding timelocks) |
| **Governance delegates** | Review all SP outputs, vote on ARFC Snapshot, vote on AIP on-chain |

### 2.1 Asset class pre-check (AAcA)

Before any listing progresses to ARFC, the candidate asset must belong to a pre-approved asset class in the [Aave Asset class Allowlist (AAcA)](https://governance.aave.com/t/arfc-aave-asset-class-allowlist-aaca/22597). If the asset's class is not on the allowlist, a separate process to evaluate and add that asset class must happen first. This prevents SPs from spending resources on due diligence for asset types that the community has not signalled appetite for.

Currently approved asset classes on Aave include (non-exhaustive): native L1/L2 tokens, major stablecoins (fiat-backed, CDP-backed), liquid staking tokens (LSTs), liquid restaking tokens (LRTs), wrapped BTC variants, yield-bearing vault tokens (ERC-4626), Pendle PT tokens, tokenised RWAs, and others. The canonical list is maintained on the [aave-dao GitHub](https://github.com/aave-dao).

### 2.2 Risk analysis

Risk service providers perform the risk analysis that determines:

- All listing parameters: supply/borrow caps, LTV, liquidation threshold, liquidation bonus, reserve factor, interest rate strategy parameters, debt ceiling, flash loan eligibility, etc.
- Whether the asset should be collateral, borrowable, or both
- eMode configuration (if applicable)
- Any special conditions or constraints

All parameters in the listing proposal are defined by risk. Other service providers do not set risk parameters.

### 2.3 Technical/security analysis

The technical service provider performs a security-focused analysis of the asset's smart contracts and dependencies. The scope and depth depends on the asset class - study previous analyses published on the governance forum for assets of the same type as reference.

This analysis is published on the governance forum as a reply to the relevant ARFC thread.

### 2.4 Oracle and pricing

The pricing strategy for a new asset is recommended by the technical SP and confirmed jointly with risk SPs. If custom price feed infrastructure is needed, the tech SP deploys it. For details on adapter types, deployment, and maintenance, see the [Aave custom price feeds bored guide](https://github.com/aave-dao/bored-guides/blob/main/aave-custom-price-feeds.md).

### 2.5 Proposal creation (AIP)

Once risk parameters are final, the oracle is decided, and the technical analysis gives a positive signal, the service provider responsible for proposal creation creates the on-chain proposal. This is done using the [aave-proposals-v3](https://github.com/aave-dao/aave-proposals-v3) repository and the Aave v3 Config Engine.

The AIP is reviewed via [Aave Seatbelt](https://github.com/aave-dao/seatbelt-gov-v3) before on-chain submission.

---

## 3. Tips for proposal creators

The following tips address common pitfalls and best practices for creating listing proposals.

### 3.1 Always use the Config Engine

Every listing proposal **must** use the [AaveV3ConfigEngine](https://github.com/aave-dao/aave-v3-origin) listing features. The Config Engine abstracts good practices when interacting with the Aave v3 protocol. Do not interact with the PoolConfigurator directly for listings.

The Config Engine handles: asset registration, interest rate strategy deployment, oracle assignment, eMode configuration, and all parameter settings in a single standardised flow.

### 3.2 Always include initial seeding

Every listing proposal must include an initial supply (seed amount) of the new asset to the pool. This is a security measure to prevent share inflation attacks on empty pools. The seed is typically sourced from the protocol team or the DAO treasury.

Follow the same approach used in recent listing proposals. The seed amount should be non-trivial relative to the asset's precision and economics but does not need to be large in dollar terms.

### 3.3 Study previous proposals and analyses

Before writing a new listing proposal, **always review multiple recent listing proposals** in the [aave-proposals-v3](https://github.com/aave-dao/aave-proposals-v3) repository, especially those for assets of the same class (e.g., if listing an LST, look at recent LST listings; if listing a stablecoin, look at recent stablecoin listings). The same applies to technical/security analyses published on the governance forum - different asset types have fundamentally different risk profiles, and studying how previous assets of the same class were analyzed is the best starting point.

Do not blindly copy-paste. Understand the rationale behind each parameter and configuration choice. For example:

- Why is a specific oracle adapter used?
- Why is the asset collateral-only vs. borrowable vs. both?
- Why are certain eMode categories assigned?
- Why is a specific seed amount chosen?

### 3.4 Include a proper test suite

Every listing proposal must include Foundry tests checking all post-conditions: correct parameters, oracle returning sensible prices, supply/borrow operations functioning, and the Config Engine interface being correctly implemented.

### 3.5 Proposal must match ARFC parameters

The on-chain proposal parameters must be exactly aligned with the parameters published in the ARFC and approved via Snapshot. Any deviation will be flagged during Seatbelt review and may result in the proposal being rejected.