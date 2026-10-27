# Testnet acceptance checklist

Status: **not executed**. No box below is satisfied by this planning document alone.

Record each result with release commit, configuration hash, environment, expected/actual outcome, and a test log or transaction link. A reproducible local test proves local behavior; a public testnet claim requires chain-998 evidence.

## Required economic behavior

- [ ] Fixed $100 contracts and long/short inverse payoff match exact reference fixtures.
- [ ] Increasing, partial closing, fully closing, and reversing preserve the correct entry basis and cash flows.
- [ ] BTC fees, positive/negative funding, margin changes, and liquidation costs reconcile in integer token units.
- [ ] One-way isolated positions cannot consume another trader's funds or unrelated free collateral.
- [ ] Splitting trades, repeated deposit/withdrawal, and funding checkpointing cannot create rounding profit.
- [ ] Reduce-only orders cannot increase or reverse exposure.
- [ ] Parameter values and differences from Binance are visible in the release specification.

## Pool and insolvency behavior

- [ ] LP share value includes open P&L, pending funding, fees, reserves, and uncollectible losses.
- [ ] LP donations, timing, or withdrawals cannot extract more than the defined entitlement.
- [ ] Exposure caps and reserves reject risk-increasing activity when capital is insufficient.
- [ ] Price-gap liquidations cannot make the system claim nonexistent assets.
- [ ] The insurance/pool/emergency waterfall is tested with forced shortfalls.
- [ ] Emergency recovery is independent of claim order and cannot use segregated free user cash.
- [ ] The largest permitted account set and settlement batch fit measured gas limits.

## Oracle, execution, and funding

- [ ] Live BTC/USD observations have recorded provenance, units, observation time, and validity checks.
- [ ] Zero, stale, replayed, out-of-order, future-dated, and materially divergent observations are rejected according to the frozen policy.
- [ ] Requests cannot be settled at a keeper-selected favorable historical price.
- [ ] Limit/TP/SL trigger and execution behavior respects deadlines, slippage, and reduce-only constraints.
- [ ] Funding uses the correct epoch, eligible position size, rate, and price; retries cannot double-charge it.
- [ ] Long keeper outages cannot erase or silently reprice historical funding.
- [ ] Oracle outage/recovery leaves balances and pending orders consistent.

## Security and automated verification

- [ ] Unit, reference-model differential, integration, and stateful invariant suites pass at the release commit.
- [ ] Reentrancy, authorization, signed-message replay where applicable, numeric bounds, and rounding attacks have dedicated cases.
- [ ] Custody and role review finds no unrestricted keeper/admin path to trader funds.
- [ ] Runtime and deployment checks reject mainnet chain 999.
- [ ] Dependencies, compiler, source revision, and licenses are recorded.
- [ ] Known limitations and any unresolved findings are explicitly listed. No unresolved finding allows incorrect settlement or unauthorized asset movement.

## Testnet release evidence

- [ ] Contracts deployed on chain 998 with a manifest of addresses, transaction hashes, roles, and configuration.
- [ ] MockBTC is clearly labeled, has 8 decimals, and is accessible to testers.
- [ ] Live-price and controlled-price deployments are distinct and identified in the UI.
- [ ] Source verification is published where supported, or matching source/compiler artifacts and verification limitations are documented.
- [ ] An accessible application connects to the intended deployment and displays confirmed contract state.
- [ ] Two unrelated trader wallets open opposing positions, close partially/fully, and withdraw through the UI.
- [ ] A third wallet supplies and withdraws pool liquidity subject to reserves.
- [ ] A controlled scenario demonstrates liquidation and another demonstrates market insolvency settlement.
- [ ] Keepers survive a restart and transaction retry without duplicate economic effects.
- [ ] Live deployment runs for at least 48 hours and crosses at least six normal funding boundaries; open positions exercise funding during the run.
- [ ] Final reconciliation accounts for all token balances, user claims, pool equity, insurance, fees, and rounding residuals.
- [ ] An operator can reproduce setup and recovery from the published runbook.

## Final demonstration script

1. Connect wallet A, obtain MockBTC and test HYPE, and deposit collateral.
2. Wallet C seeds the liquidity pool; record its initial share value.
3. Wallet A opens a long; wallet B independently opens a short. Record quantities, fills, fees, and margin.
4. Partially close A's position and increase B's; compare results with the reference model.
5. Submit and execute/cancel representative limit and TP/SL orders.
6. Hold positions through normal funding; show the BTC debit/credit and pool offset.
7. In the controlled-price market, move the price to trigger liquidation and reconcile the loss/reward.
8. Interrupt price updates and restart a keeper; demonstrate the specified restrictions and recovery.
9. Close positions and withdraw collateral; redeem eligible LP shares and reconcile final balances.
10. In a separate controlled scenario, force insolvency and demonstrate deterministic settlement recovery.

## Completion statement template

```text
Release commit:
Configuration hash:
Chain ID:
Live-price market address:
Controlled-price market address:
Collateral token address and decimals:
Oracle source and trust assumptions:
Application URL:
Deployment/verification records:
Test report:
Observation window and funding epochs:
Final reconciliation:
Known differences from Binance:
Remaining limitations:
Acceptance result: PASS / INCOMPLETE
```

Only mark the testnet target complete when all required checks have evidence. This result establishes a testnet prototype, not a mainnet security certification.
