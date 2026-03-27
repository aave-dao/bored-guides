# Aave v3 network evaluation procedure

**Author**: [BGD Labs](https://bgdlabs.com/)

---

> **Disclaimer**: This guide does NOT guarantee 100% correctness or completeness. Users MUST understand the context of application and have sufficient technical and product expertise. As a contributor and/or maintainer, always act defensively, whether trying to understand the system, testing new developments, or involving external security parties and procedures.
> 

## 1. Introduction

Whenever a new network is proposed as a candidate for deploying the Aave protocol, an independent technical/infrastructure evaluation is produced and published on the Aave governance forum. This evaluation provides the community with an informed assessment of the network's suitability from a technical standpoint, covering areas such as consensus, bridge security, tooling, and governance infrastructure compatibility. The scope and structure of these evaluations are based on the [Technical evaluation framework of a network](https://governance.aave.com/t/bgd-aave-v3-deployments-technical-evaluation-framework-of-a-network/10237) post

## 2. Governance lifecycle and participants

1. **Forum proposal.** A community member, the network team, or a growth/bizdev service provider posts a proposal requesting Aave deployment on network X. Community discussion happens on the forum thread.
2. **Temp check on Snapshot.** The community votes on whether to proceed with the network expansion.
3. **ARFC stage.** After a positive temp check, the proposal moves to ARFC. At this stage, service providers perform their evaluations: the technical service provider produces the infrastructure/technical evaluation and publishes it on the forum, risk service providers provide risk parameters and asset recommendations. These inputs inform the ARFC discussion and Snapshot vote.
4. **AIP.** If everything aligns, the deployment proceeds through on-chain governance vote and execution.

| Participant | Role |
| --- | --- |
| **Network team** | Provides documentation and answers technical questions |
| **Growth / bizdev service provider** | Authors and publishes governance proposals (TEMP CHECK, ARFC), manages the proposal lifecycle through Snapshot votes and AIP creation, plans and manages liquidity incentive programs, coordinates with network teams on business terms |
| **Technical service provider** | Produces the independent technical/infrastructure evaluation. Performs [asset technical analysis](https://github.com/aave-dao/bored-guides/blob/main/aave-v3-asset-listings.md) for assets proposed for listing on the new deployment. Works on network technical evaluation. Prepares [a.DI/Governance infrastructure deployment](https://github.com/aave-dao/bored-guides/blob/main/adi-governance-network-expansions.md) and activation AIPs |
| **Risk service providers** | Risk analysis, asset parameter recommendations, market risk assessment |
| **Aave Governance (token holders)** | Final decision via Snapshots and on-chain AIP vote |

## 3. Tips

- **Use the [evaluation framework](https://governance.aave.com/t/bgd-aave-v3-deployments-technical-evaluation-framework-of-a-network/10237) as a starting point.** It covers the baseline areas to assess, but every network is different. Novel architectures, unusual bridge designs, or non-standard EVM compatibility may require investigating areas not covered by the framework. Published evaluations on the [Aave governance forum](https://governance.aave.com/c/development/26) demonstrate how the framework has been applied and extended. Reviewing prior evaluations of similar networks (e.g., other OP Stack L2s, other zkEVMs) is the fastest way to calibrate.
- **Assess [a.DI](https://github.com/aave-dao/aave-delivery-infrastructure) compatibility.** Aave governance must be able to operate on the network cross-chain. If no viable communication path exists (native bridge or supported third-party providers), the deployment is blocked. See the [a.DI/Governance network expansion](https://github.com/aave-dao/bored-guides/blob/main/adi-governance-network-expansions.md) guide for the deployment process
- **Verify EVM compatibility and all required infrastructure independently.** Contract deployment, on-chain reads, explorer verification, and everything else Aave depends on should be tested directly on the network. Missing infrastructure can block the deployment entirely.
- **Assets require technical analysis even if already listed on other Aave deployments.** A new network deployment entails different bridge implementations, distinct token contract addresses, and potentially varying trust assumptions for the same underlying asset. The [asset listing procedure](https://github.com/aave-dao/bored-guides/blob/main/aave-v3-asset-listings.md) guide covers this process.
- **Prioritize clarity and specificity.** Spend time on areas where the network deviates from well-known configurations - that is, where the evaluation adds the most value. Be concrete about trust assumptions (who controls what, with what thresholds and delays), include links so the reader can verify, and surface the most material findings early in the post.
- **Consider a staged launch when conditions warrant it.** If the network is relatively new, liquidity has not yet been established, or certain infrastructure components are still maturing, it may be prudent to recommend launching in two stages - an initial deployment with restricted parameters, followed by a full activation once the ecosystem is more settled.