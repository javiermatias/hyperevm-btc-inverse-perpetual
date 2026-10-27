# Proposed BTC inverse market specification

Version: planning draft, 2026-10-27. Values marked **proposed** are design choices for this project, not statements of Binance's current trading parameters. Stage 0 freezes them; Stage 1 validates their economic effects.

## Product and parity target

Binance documents cryptocurrency settlement and a $100 contract multiplier for BTC COIN-M. Its inverse P&L depends on reciprocal entry and exit prices. Those are the core behaviors to reproduce. [Binance contract overview](https://www.binance.com/en-AE/support/faq/detail/85eac2bba0b342819122dc9bd4745e9b), [Binance calculator](https://www.binance.com/en/support/faq/detail/360036498511)

| Property | Testnet target | Relationship to COIN-M |
| --- | --- | --- |
| Market | BTCUSD-INVERSE perpetual | Same underlying economic exposure |
| Contract face value | $100; whole-contract quantities | Matches the documented BTC multiplier |
| Collateral/settlement | MockBTC, 8 decimals | BTC units; mock asset has no real backing |
| Position mode | One-way; one net position per account | Initial supported mode |
| Margin | Isolated position collateral | Initial supported mode |
| Leverage | Proposed 1x–5x | Deliberately small testnet range |
| Maintenance rate | Proposed flat 2.5%, plus specified close-cost reserve | Project parameter; not Binance tier replication |
| Fees | Proposed 5 basis points per executed trade, charged in BTC | Same settlement currency; custom rate |
| Funding | BTC transfers at scheduled epochs; main demo every 8 hours | Similar schedule/units; local rate formation |
| Liquidity | Pool counterparty with explicit reserve and exposure limits | Custom execution model |
| Profit handling | Inverse P&L without an arbitrary multiple-of-margin profit cap | Subject to liquidation and defined insolvency settlement |
| Orders | Market requests, limits, reduce-only, partial close, TP/SL | Basic trading lifecycle |
| Liquidation | Mark-based health; initial full-liquidation implementation | Tiered/partial liquidation is a later refinement |

Exact Binance mark prices, funding rates, fees, liquidation thresholds, matching behavior, ADL, and real BTC redemption are outside the parity claim. Numerical payoff parity must be tested with identical hypothetical fills and costs.

## Units and position arithmetic

Use these mathematical quantities before introducing Solidity fixed-point representations:

- `q`: signed integer contract count; positive is long, negative is short.
- `C = 100`: USD face value per contract.
- `N = abs(q) × C`: positive USD notional.
- `s = sign(q)`: +1 for a long, -1 for a short.
- `E`, `X`, `M`: entry, execution/exit, and mark prices in USD per BTC.
- `B`: BTC cash allocated to the isolated position, after already-settled cash flows.

The following is the project's accounting specification using inverse-contract mathematics. Implement units and rounding explicitly; do not use binary floating-point arithmetic for reference balances.

```text
position_value_BTC(M) = N / M
unrealized_PnL_BTC(M) = s × N × (1/E − 1/M)
realized_PnL_BTC(X)   = s × N_closed × (1/E − 1/X)
initial_margin_BTC   = N / (E × leverage)
trade_fee_BTC        = N_executed / execution_price × fee_rate
```

Initial margin is not the total required deposit: add entry fees and the defined close-cost buffer. After opening or increasing, enforce both the chosen leverage requirement and maintenance safety on the resulting position at the current valid mark.

For increases in the same direction, the effective entry price is a notional-weighted harmonic mean:

```text
E_new = (N_old + N_added) / (N_old/E_old + N_added/E_added)
```

Maintain an adequately precise entry accumulator rather than repeatedly rounding a displayed entry price. A partial close realizes only the closed portion; the surviving position keeps its entry basis. A reversal closes the old position first and opens only the residual quantity at the new fill. Fees apply to the actual executed amount once.

`reduceOnly` can reduce a position to zero, but cannot increase its absolute size or reverse its direction. Settle due funding before any size change.

### Exact reference fixtures

Illustrative prices only. Gross P&L excludes fees and funding.

| Case | Expected result |
| --- | --- |
| Long 100 contracts, entry $100,000, exit $110,000 | `+1/110 BTC`, approximately `+0.009090909091` |
| Long 100 contracts, entry $100,000, exit $90,000 | `−1/90 BTC`, approximately `−0.011111111111` |
| Short 100 contracts with the same entry/exits | Exact opposite of the long's gross P&L |
| 100 contracts at $100,000, leverage 5x | `0.02 BTC` initial margin, before costs |
| Add 100 contracts at $80,000 to 100 at $100,000 | Entry becomes `$800,000/9`, approximately `$88,888.888889` |
| Close 40 of the original 100-contract long at $110,000 | `+1/275 BTC`; 60 contracts remain at $100,000 entry |
| $10,000 executed at $100,000, fee rate 0.0005 | `0.00005 BTC` fee, or 5,000 satoshis |
| $10,000 long, funding mark $100,000, rate +0.0001 | Trader pays `0.00001 BTC`, or 1,000 satoshis |

Fixtures must be checked at high precision and after the defined token-rounding rule. Reject zero/negative prices and guard multiplication, division, and extreme reciprocal values.

## Margin and liquidation

Proposed initial account equations:

```text
equity_BTC = allocated_cash_BTC + unrealized_PnL_BTC − unsettled_funding_due_BTC
maintenance_BTC = maintenance_rate × N / M
liquidatable = equity_BTC <= maintenance_BTC + liquidation_close_cost_buffer_BTC
```

Funding due is signed: a credit increases equity. Specify the buffer, liquidator reward, and boundary rounding before implementing this predicate. A displayed liquidation price must come from these rules and current funding/collateral; it is an estimate that can move.

Adding/removing margin changes position collateral without changing contract count. Removal is permitted only after due funding is applied and the remaining position meets the specified margin requirement. Free wallet collateral is not automatically swept into an isolated position.

Liquidation uses a valid mark for eligibility and the documented close-price policy for realization. The policy must include stale-price behavior and any liquidation spread; a keeper cannot supply an arbitrary favorable price. No liquidator reward may be minted or exceed the assets available under the waterfall.

## Funding

Binance describes periodic funding, a default eight-hour schedule, premium/interest components, and adjustable parameters. The project will snapshot current reference settings, while using its own pool's pricing observations. [Binance funding documentation](https://www.binance.com/en/support/faq/detail/360033525031)

Project accounting for an epoch `i`:

```text
trader_cash_flow_BTC_i = −signed_USD_notional × funding_rate_i / funding_mark_i
pool_cash_flow_BTC_i   = −sum(trader_cash_flow_BTC_i)
```

Positive rates charge longs and credit shorts. The pool receives or pays the unmatched net amount. Funding redistributes existing claims; it does not create tokens.

Use a cumulative BTC-per-USD funding index with per-position checkpoints. Determine epoch eligibility before orders at the boundary change quantities. An interval of inactivity must not erase funding, and delayed processing must use the frozen epoch rate and validated historical price, not a later spot price.

Stage 0 must freeze the premium sampling/weighting rule, rate cap/floor, interest component, and missing-observation policy. For the initial pool, derive a bounded premium from its executable bid/ask quotes. Record this as a custom rate model; the absence of a Binance-like order book prevents exact funding-rate replication.

## BTC precision

- Token transfers are integer satoshis; proposed price representation is 8 decimal places in USD/BTC.
- Use higher internal precision for inverse entry and funding accumulators. Prove numeric bounds for permitted prices, contract counts, and elapsed epochs.
- Specify directional rounding per operation: trader credits must not exceed the calculated entitlement; required deposits and charges must not be understated.
- Carry or explicitly allocate residual dust in a ledger. Repeated splitting, closing, or funding checkpointing must not create value.
- Internal cash movements must conserve total assets. Fees and insurance are explicit destinations, not unexplained balance differences.

## Pool economics and LP shares

LPs supply counterparty capital. Trader gains reduce pool equity; collectible trader losses increase it. The pool's exposure must be modeled in inverse units; copying a linear market's hedge or risk formula is insufficient.

For a snapshot price, value every open position and pending funding liability before pricing LP shares. Do not recognize an insolvent trader's negative equity as fully collectible pool income. LP deposits and withdrawals must use the same valuation rules and reserve checks.

Admission checks include available payout liquidity, gross and net exposure limits, price-band stress loss, and existing obligations. Define reserves before accepting additional positions or LP withdrawals. Exposure limits and stress tests reduce risk; they do not guarantee solvency at all prices. Before liquidation, an inverse long's theoretical BTC loss grows without bound as BTC's USD price approaches zero; the corresponding short's BTC gain grows without bound. A pool facing profitable shorts can therefore owe more BTC than its finite reserves if liquidation and other risk controls fail.

Proposed loss process:

1. Realize a losing position against its allocated margin.
2. Charge any uncovered liquidation loss to the designated insurance reserve, then available LP risk capital.
3. If obligations still exceed the market's available assets, freeze new exposure and LP redemptions and enter emergency settlement.
4. Snapshot the prescribed settlement price, obligations, and available market assets. Apply deterministic proportional recovery to affected trading claims. Never give earlier claimants a better recovery just because they withdraw first.

Separately held free trader collateral is excluded from counterparty capital. Normal trading P&L and fees must remain distinct from emergency loss allocation. This mechanism is a project design, not a claim to reproduce Binance's insurance or ADL system.

## Items to freeze before implementation

- [ ] Exact fixed-point units, numeric bounds, rounding, and dust allocation.
- [ ] Fee, margin, liquidation-buffer, and liquidator-reward parameters.
- [ ] Index source, mark rule, execution quote curve, spread/skew bounds, and observation age limits.
- [ ] Order timing and cancellation rules with adversarial examples.
- [ ] Funding derivation, epochs, missing-data handling, and processing order.
- [ ] Gross/net caps, reserve formula, LP withdrawal rules, and initial pool capitalization.
- [ ] Emergency price selection, recovery priority, and proof that claim order cannot affect payouts.
