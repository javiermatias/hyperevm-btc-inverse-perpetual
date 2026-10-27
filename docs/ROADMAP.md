# Staged implementation roadmap

Created: 2026-10-27. Target: a demonstrable HyperEVM testnet BTC inverse perpetual market.

All stages below are **not started**. Completing this planning package does not imply that the market has been built or validated. Advance after the preceding gate passes, and record results against the [acceptance checklist](TESTNET_ACCEPTANCE.md).

## Stage 0 — Freeze the rules and verify dependencies

**Purpose:** make the financial contract and operational assumptions precise enough to implement.

Tasks:

- Capture a dated Binance `BTCUSD_PERP` reference: contract size, tick/lot rules, available margin modes, funding interval, and relevant risk parameters. Record inaccessible or account-dependent fields instead of inventing them.
- Adopt the [market specification](MARKET_SPEC.md), explicitly recording the testnet deviations. Freeze units, rounding, leverage tiers, fee rates, funding rule, execution timing, and liquidation waterfall in a versioned configuration.
- Specify the binding order lifecycle: request, price observation, execution, rejection, expiry, and cancellation. Resolve how delayed execution and funding boundaries interact. A trader or keeper must not choose among already-known prices to extract value from the pool.
- Select the oracle route described in [Architecture](ARCHITECTURE.md). Probe chain 998 for the actual BTC feed, decimals, source timestamps, update cadence, round history, and divergence behavior. A readable price alone does not prove freshness.
- Verify RPC access, test HYPE availability, deployment gas limits, and usable explorer links. Use MockBTC for collateral, independent of whether bridged BTC exists on testnet.
- Inspect candidate dependencies at exact commit hashes and record licenses. Check D8X's position conventions against our fixed-USD inverse payoff before reusing any financial logic.
- Choose initial pool capital, per-account/global gross exposure caps, net exposure caps, maximum open accounts, and reserve rules. These remain provisional until simulation.
- Decide the complete protocol for insufficient oracle observations and missed funding epochs. Do not silently fabricate a historical observation or substitute today's price.

**Deliverables:** versioned market configuration, decision log, dependency lock/attribution record, oracle probe report, and a testnet setup checklist.

**Gate:** no unresolved decision changes P&L, collateral ownership, funding eligibility, execution price, liquidation, or LP withdrawal rights. The live oracle route and test-gas route have been demonstrated. If external access is unavailable, continue local work and mark testnet deployment as pending.

## Stage 1 — Build the reference engine and simulate the pool

**Depends on:** Stage 0 financial rules. Local work can continue while testnet infrastructure is being resolved.

Tasks:

- Implement an independent reference model using exact rational arithmetic or high-precision decimal arithmetic. Keep it independent of the Solidity implementation.
- Model deposits, collateral allocation, long/short entry, increasing a position, partial/full closes, reversals, fees, funding, margin changes, and liquidation.
- Reproduce the numerical fixtures in [Market specification](MARKET_SPEC.md). Add multi-fill, funding-boundary, rounding, and bankruptcy fixtures.
- Model the pool as the counterparty to aggregate trader exposure, including open P&L, pending funding, uncollectible trader debt, fees, insurance, and LP share value.
- Simulate one-sided demand, rapid rises and falls, gaps of 20%/50%/80%, a major upward jump, oracle outages, delayed liquidations, and simultaneous withdrawal attempts. These are test scenarios, not probability forecasts.
- Compare behavior with and without risk limits. Find the price paths that cause reserve exhaustion; do not assume leverage limits guarantee solvency.
- Calibrate initial caps and reserve policy. Reject the configuration if the pool cannot support the intended demonstration under the agreed scenario suite.

**Deliverables:** reference engine, deterministic fixtures, simulation runner, plots/tables of pool equity and trader outcomes, and a documented risk configuration.

**Gate:** inverse P&L matches exact fixtures; ledger conservation holds; no account can improve its balance through splitting trades or rounding cycles; pool failure conditions and settlement behavior are reproducible.

## Stage 2 — Implement collateral, positions, and pool accounting

**Depends on:** Stage 1.

Tasks:

- Bootstrap Solidity/Foundry contracts with pinned dependencies and a pinned compiler. Add CI for build, unit tests, differential tests, and invariants.
- Implement MockBTC, the BTC vault, inverse arithmetic, isolated margin, and one net position per trader.
- Implement allocated and free collateral, BTC trading fees, partial realization, entry-price aggregation, and reduce-only checks.
- Implement LP deposits/share issuance, share valuation including open liabilities, reserved capital, and constrained withdrawals. Prevent donation/inflation and stale-NAV attacks.
- Enforce minimum trade size, precision bounds, account/open-order caps, and role restrictions.
- Use testnet-only deployment guards: remote deployments must target chain 998; local tests may use 31337. Reject chain 999.
- Provide events sufficient to reconstruct every cash transfer and position change independently.

**Deliverables:** contract modules, deployment scripts for local use, ABI artifacts, event schema, and tests against the reference engine.

**Gate:** deposits and withdrawals reconcile to token balances; custody access is constrained; long/short and partial-close fixtures pass; LP accounting recognizes liabilities before distributions.

## Stage 3 — Implement pricing, order execution, and funding

**Depends on:** Stage 2 and Stage 0 oracle/execution decisions.

Tasks:

- Implement the production-shaped oracle interface and a separate controlled-price test adapter. Normalize units and validate timestamps, positive prices, source identity, and monotonic rounds.
- Separate index, mark, and execution price. Implement the frozen spread/skew model and validate requested slippage bounds on-chain.
- Implement binding market requests and conditional orders. Include nonces, deadlines, cancellation rules, reduce-only behavior, and ordering around funding boundaries.
- Implement funding in BTC with an epoch accumulator. Settle funding before changing position size, collateral availability, or LP entitlement.
- Use an 8-hour schedule for the main testnet demonstration. Use an explicitly labeled accelerated configuration only for automated demonstrations.
- Derive funding from this market's pricing rule; do not import another exchange's realized funding rate as if it were this pool's rate.
- Define behavior for stale prices, missing observations, downtime, recovery, and retries. Keep operations bounded so a long outage cannot create an unexecutable catch-up loop.

**Deliverables:** oracle adapters, order state machine, pricing module, funding engine, and boundary/outage tests.

**Gate:** stale or replayed data cannot authorize trades; neither traders nor keepers can select favorable historical prices; positive/negative funding transfers reconcile; missing epochs cannot be skipped to avoid obligations.

## Stage 4 — Implement liquidations and validate loss containment

**Depends on:** Stage 3.

Tasks:

- Implement margin-health calculations, close-fee reserves, liquidation eligibility, liquidation incentives, and full liquidation for the initial small-account configuration.
- Charge loss against the position's isolated collateral; implement the documented insurance/pool/market-settlement waterfall for any shortfall.
- Implement market states for normal operation, restricted operation, and emergency settlement. Emergency settlement must use a documented price rule and fixed entitlements, independent of withdrawal order.
- Enforce gross and net exposure limits, pool reserves, and LP withdrawal restrictions under open P&L. Ensure that unrelated free trader collateral cannot fund pool losses.
- Test flash-funded trades, repeated tiny trades, reentrancy, price manipulation, stale observations, pending-order races, price gaps, failed keepers, and mass withdrawals.
- Run stateful fuzzing and differential sequences against Stage 1. Measure the gas cost of the largest allowed account set and settlement batches.

**Deliverables:** liquidation and settlement modules, adversarial tests, invariant results, and a risk review with reproduced failures and fixes.

**Gate:** liquidations and settlement conserve assets; insolvent accounts cannot withdraw fabricated value; insufficient collateral triggers the specified process; no known unresolved issue allows unauthorized asset movement or incorrect economic settlement.

## Stage 5 — Build services and the trading interface

**Depends on:** stable Stage 3 interfaces; final sign-off depends on Stage 4.

Tasks:

- Build keeper services for oracle updates where required, order execution, funding checkpoints, and liquidations. Persist transaction state and handle duplicate jobs, dropped transactions, nonce conflicts, reverts, and restarts.
- Build a minimal wallet interface: connect to chain 998, claim MockBTC, approve/deposit, choose long/short and leverage, preview fees/margin, submit/cancel orders, close partially/fully, and withdraw.
- Add basic limit orders and reduce-only TP/SL requests. Show that trigger activation does not guarantee a fill price.
- Display BTC equity, realized/unrealized P&L, index/mark/execution quotes, funding countdown, margin health, and pool availability. Calculate liquidation estimates from the actual engine.
- Add LP deposit/withdrawal views, reserve utilization, and outstanding withdrawal state.
- Label mock collateral, controlled/live price mode, stale data, and restricted-market status. Read on-chain balances as the authority; the UI must not invent success before transaction confirmation.
- Add monitoring for oracle age, keeper lag, funding backlog, pool risk, and failed transactions.

**Deliverables:** application, keeper service, event index/read layer, environment templates without secrets, and operator runbooks.

**Gate:** two unrelated trader wallets and an LP can complete the complete lifecycle through the interface. Restarted keepers do not duplicate economic actions. UI values reconcile with contracts and the reference model.

## Stage 6 — Run an integrated deployment rehearsal

**Depends on:** Stages 0–5.

Tasks:

- Run local integration tests using realistic token decimals, fresh and stale prices, delays, and RPC failures. Use HyperCore simulation only for components that actually depend on HyperCore.
- Freeze a release commit and parameter file; produce compiler metadata, dependency revisions, ABIs, and expected deployment addresses where deterministically available.
- Estimate deployment and runtime gas against current testnet limits. Split modules or use documented larger-block deployment handling if necessary, then restore normal transaction settings.
- Rehearse fresh deployment, initialization, liquidity seeding, role handover, service startup, pause/recovery, and controlled emergency settlement.
- Run every applicable item in [Testnet acceptance](TESTNET_ACCEPTANCE.md) locally. Record limitations instead of treating mocks as evidence of live integrations.

**Deliverables:** reproducible release candidate, passing test report, deployment manifest template, and rehearsed runbook.

**Gate:** a clean environment can reproduce the deployment and scenario suite. Required test-gas access and oracle availability are confirmed immediately before testnet deployment.

## Stage 7 — Deploy and demonstrate on HyperEVM testnet

**Depends on:** Stage 6.

Tasks:

- Deploy the live-price release on chain 998. Deploy the controlled-price scenario market separately and label it clearly.
- Record chain ID, addresses, transaction hashes, source commit, configuration hash, roles, token decimals, oracle provenance, and application URL. Verify source on a compatible explorer when supported; otherwise publish matching source and compiler artifacts and record the limitation.
- Seed MockBTC liquidity. Demonstrate long, short, partial close, margin changes, fees, positive/negative funding scenarios, liquidation, LP withdrawals, and emergency settlement.
- Run the live-price system for **at least 48 hours and six normal funding boundaries**, keeping positions open across enough boundaries to exercise real accounting. Extend the observation period if needed to meet both conditions.
- Demonstrate keeper restart and oracle interruption/recovery without silently forgiving funding or changing account entitlements.
- Publish the completed acceptance report with evidence links and remaining differences from Binance.

**Deliverables:** functioning testnet market, accessible interface, running services, deployment manifest, demonstration instructions, and acceptance report.

**Gate:** all required acceptance items pass. A live-price outage, missing gas, or unavailable wallet may block deployment, but does not turn a local prototype into a completed testnet release.

## Extensions after the first accepted testnet

| Extension | Additional proof required |
| --- | --- |
| Partial liquidation and richer margin tiers | Reference fixtures and stress tests for tier transitions |
| Cross margin or simultaneous hedge mode | New account-risk and position-netting specification |
| Real BTC representation | Verified token, backing/redemption, decimals, transfer paths, and test coverage |
| HyperCore hedging | Correct inverse-exposure hedge, confirmed fills, basis/funding costs, separate USDC margin, and delayed-action recovery |
| Order-book execution | Matching rules, maker/taker accounting, liquidity, and mark-price design |
| Mainnet | Separate launch decision, external review, operational readiness, and a new acceptance scope |

These extensions must not be prerequisites for completing the agreed testnet MVP. The target remains the economics and user experience of BTC COIN-M within the explicitly documented initial scope.
