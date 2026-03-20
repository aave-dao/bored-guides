# Custom price feeds

**Author**: [BGD Labs](https://bgdlabs.com/)

---

> **Disclaimer**: This guide does NOT guarantee 100% correctness or completeness. Users MUST understand the context of application and have sufficient technical and product expertise. As a contributor and/or maintainer, always act defensively, whether trying to understand the system, testing new developments, or involving external security parties and procedures.
> 

## 1. Introduction

Aave pools require price feeds for every listed asset. In many cases, no single external feed covers the need - the price may need to be capped against exchange rate manipulation, composed from multiple sources, discounted for PT tokens, wrapped for SVR integration, or computed from on-chain state when no external feed exists.

These custom price feed adapters sit between the raw price sources and the Aave Oracle, transforming or constraining prices before they reach the pool. They are deployed and maintained as part of the asset lifecycle across Aave V3 instances.

This guide covers the procedures for listing new adapters, migrating existing oracle sources, and managing adapter parameters. 

## 2. Adapter types and repository structure

This repository ([aave-dao/aave-price-feeds](https://github.com/aave-dao/aave-price-feeds)) contains all adapter contracts, deployment scripts, and tests. For detailed mechanics, formulas, parameters, and constraints of each adapter type, see the in-repo documentation:

- [Core adapters](https://github.com/aave-dao/aave-price-feeds/blob/main/src/contracts/README.md) - rate-capped (`PriceCapAdapterBase`), stable (`PriceCapAdapterStable`), EUR stable (`EURPriceCapAdapterStable`), Pendle PT (`PendlePriceCapAdapter`), synchronicity adapters
- [Misc adapters](https://github.com/aave-dao/aave-price-feeds/blob/main/src/contracts/misc-adapters/README.md) - `FixedPriceAdapter`, `OneUSDFixedAdapter`, `DiscountedMKRSKYAdapter`
- [How to add new adapters](https://github.com/aave-dao/aave-price-feeds/blob/main/how-to.md) - step-by-step implementation guide

## 3. Procedures

### 3.1 Listing a new custom price feed adapter

### Step 1: Scope the need

1. Risk and technical service providers determine why a custom price feed is needed instead of a plain Chainlink feed.
2. They choose the adapter family: Rate-Capped / Stable / PT / Custom.
3. If a required Chainlink price feed or exchange-rate feed does not yet exist, it should be requested early, since adapter work depends on it.

### Step 2: Define the inputs

1. Risk providers provide the cap parameters.
2. Technical service providers locate the deployed Chainlink feed and ratio provider addresses on the target network.
3. For Rate-Capped adapters, technical service providers compute `snapshotRatio` and `snapshotTimestamp` using `GetExchangeRatesTest` at an appropriate historical block.
4. For Custom adapters, the pricing formula, decimals assumptions, and admin/update path should be documented explicitly before implementation.

### Step 3: Implement and validate

1. Reuse an existing generic adapter whenever possible (`CLRatePriceCapAdapter`, `PriceCapAdapterStable`, `PendlePriceCapAdapter`). If none fit, implement a new adapter and supporting interface(s).
2. Add deployment code to the relevant network script.
3. Add or extend adapter tests.
4. Run retrospective validation and generate reports.
5. If the adapter introduces new custom logic, additional review or security assessment may be required before governance execution.

### Step 4: Deploy and integrate

1. Deploy the adapter contract and verify it returns expected prices on-chain.
2. Include the adapter in the relevant governance proposal (asset onboarding, network activation, or standalone oracle update AIP).
3. After execution, verify that the oracle source has been updated correctly on-chain.

See [how-to guide](https://github.com/aave-dao/aave-price-feeds/blob/main/how-to.md) for detailed implementation instructions.

### 3.2 Replacing an existing oracle source/adapter

A migration replaces the contract registered in the AaveOracle contract as the source of a listed asset. Typical reasons include:

- **Asset pricing model changes** - the asset becomes yield-bearing, rebasing, share-based, or otherwise needs exchange-rate-aware pricing
- **Base price source changes** - the base Chainlink feed used by the adapter is no longer the preferred source
- **Exchange-rate source changes** - the ratio source changes from the native rate to the Chainlink feed, or vice versa
- **Feed deprecation or replacement** - Chainlink retires a feed or deploys a newer one
- **Adapter logic changes** - the pricing formula or risk assumptions require a different contract, not just different parameters
- **SVR activation** - the base Chainlink feed is replaced by its SVR (Smart Value Recapture) equivalent to enable MEV recapture from liquidations

### Migration flow

1. Confirm that parameter maintenance is insufficient and that the Oracle source itself must change.
2. Define the target pricing topology (new base feed, ratio source, adapter type, or formula).
3. Follow the same implementation and validation flow as for a new listing (parameter collection, snapshot calculation, implementation, tests, retrospective reports).
4. Deploy the new adapter contract.
5. Submit a governance proposal that updates the asset source in Aave Oracle to the new adapter.
6. After execution, verify that the new adapter is the active oracle source and behaves as expected on-chain.

### 3.3 Parameter management

After deployment, adapter parameters need ongoing maintenance to stay aligned with reality. On-chain, `setCapParameters` / `setPriceCap` / `setDiscountRatePerYear` are callable by any address with `RiskAdmin` or `PoolAdmin` role in the pool's ACL Manager. In practice, these calls come through the Risk Agents infrastructure - either manually (Risk Pilot submits, Risk Guardian can cancel during timelock) or via automated Risk Oracles.

Both paths are subject to on-chain constraints (max change per parameter, minimum interval between updates) enforced by the risk infrastructure contracts. Changes exceeding these constraints, or changes to the constraint framework itself, require governance (AIP).

### 3.4 Testing

Retrospective tests and reports are required for every new listing and migration. See [how-to guide](https://github.com/aave-dao/aave-price-feeds/blob/main/how-to.md) and the test base classes (`BaseTest`, `CLAdapterBaseTest`, `BaseStableTest`) for details.

## 4. General tips

- If a custom price feed adapter already exists for the same asset on the same network, reuse it rather than deploying a new one - for example, sharing adapters between Core and Prime instances.
- When requesting a new Chainlink feed, verify its correctness and assess feed risk (update frequency, deviation threshold, data sources) before building an adapter on top of it.
- Output decimals: all adapters inherit decimals from their base Chainlink feed. **Aave requires 8-decimal price feeds**. Chainlink has started deploying feeds with 18 decimals on some networks - with the current Aave V3 and Aave Oracle architecture, only 8-decimal feeds should be used as base aggregators.
- `baseAggregator` is not necessarily a raw Chainlink feed - it can be another adapter. Adapters can be composed into chains.
- Access control: price cap adapters use ACL Manager - RiskAdmin or PoolAdmin can update parameters.
- Always keep in mind the synchronicity of price updates on correlated assets. For example, changing the ETH/USD feed for WETH in the same Aave instance to SVR, while keeping wstETH using wstETH/ETH + ETH/USD (non-SVR) is a major problem to the system: while SVR ETH/USD and non-SVR ETH/USD will usually report the same price, they are independent feeds technically, so if price of ETH decreases, one would update before the other, potentially causing liquidations that should never have happened.
- SVR feeds are standard Chainlink feeds from the adapter's perspective - same interface, same decimals. SVR activation is a base aggregator swap: the adapter is redeployed with the SVR feed address, keeping all other parameters and ratio sources identical.
- Every SVR activation requires deploying an SvrOracleSteward per pool instance. The steward allows the Protocol Guardian to instantly roll back any SVR-based oracle to its pre-SVR predecessor. The governance payload grants AssetListingAdmin to the steward and calls enableSvrOracles() with the asset-to-oracle mapping.