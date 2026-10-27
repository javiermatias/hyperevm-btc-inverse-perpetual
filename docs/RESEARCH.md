# Research, reuse candidates, and decisions

Research reviewed: 2026-10-26/27. Plan authored: 2026-10-27.

The selected direction is a custom HyperEVM market with independently controlled trader positions and a BTC counterparty pool. Financial rules proposed elsewhere in this package are project decisions. External facts below describe the sources reviewed; live availability and parameters require Stage 0 checks.

## Official references

| Source | What it establishes for this plan |
| --- | --- |
| [Binance COIN-M overview](https://www.binance.com/en-AE/support/faq/detail/85eac2bba0b342819122dc9bd4745e9b) | Coin settlement and BTC contract face value |
| [Binance futures calculator](https://www.binance.com/en/support/faq/detail/360036498511) | Inverse P&L and initial-margin reference |
| [Binance COIN-M margin](https://www.binance.com/en/support/faq/detail/be2c7d9d95b04a7e8044ed02dd7dfe5c) | Leverage/margin concepts; live tiers need a dated capture |
| [Binance funding](https://www.binance.com/en/support/faq/detail/360033525031) | Funding conventions and the fact that intervals/parameters can change |
| [Binance COIN-M market-data API](https://developers.binance.com/en/docs/catalog/core-trading-derivatives-trading-coin-m-futures/api/rest-api/market-data) | Exchange information, mark/index data, and funding information for the Stage 0 snapshot |
| [HyperEVM](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/hyperevm) | EVM application environment and network configuration |
| [HyperCore integration](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/hyperevm/interacting-with-hypercore) | Precompiles and CoreWriter interfaces |
| [Dual-block architecture](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/hyperevm/dual-block-architecture) | Larger deployment transactions may require different block routing; inspect current limits |
| [HyperEVM tools/faucets](https://hyperliquid.gitbook.io/hyperliquid-docs/builder-tools/hyperevm-tools) | Candidate sources of test HYPE and explorers; availability was not exercised |

## Why custom HyperEVM

HIP-3 allows eligible quote assets as market collateral. The published permissionless quote-asset requirements include a USD peg. We therefore did not verify a permissionless BTC-settled HIP-3 route. BTC support in Portfolio Margin does not by itself change a perp's settlement denomination. [HIP-3 specification](https://hyperliquid.gitbook.io/hyperliquid-docs/hyperliquid-improvement-proposals-hips/hip-3-builder-deployed-perpetuals), [Quote-asset requirements](https://hyperliquid.gitbook.io/hyperliquid-docs/hypercore/permissionless-spot-quote-assets), [Portfolio Margin](https://hyperliquid.gitbook.io/hyperliquid-docs/trading/portfolio-margin)

A custom EVM implementation is an engineering approach supported by programmable contracts. It requires building and validating the market logic ourselves. It is not an existing native BTC COIN-M feature or a guarantee of economic viability.

## GitHub candidates

### psyto/hyperevm-perp

[Repository](https://github.com/psyto/hyperevm-perp)

The repository describes a USDC-margined pool market with CoreWriter hedging, an Apache-2.0 license, 44 tests, and testnet demonstrations. It explicitly identifies itself as an unaudited learning project. Its documented restrictions include a profit cap, a trusted hedge keeper, one liquidity provider, and absent trader funding.

**Use:** execution/recovery reference. It is not a drop-in BTC inverse market. Test counts and deployment outcomes are author-reported; they were not independently rerun in this research. Inspect the complete source, dependencies, and revision before reusing code.

### D8-X/perpetual-simulation

[Repository](https://github.com/D8-X/perpetual-simulation) · [Model source](https://github.com/D8-X/perpetual-simulation/blob/main/perpetual.py) · [D8X contract descriptions](https://d8x.gitbook.io/d8x/contribute-and-earn/traders)

The MIT-licensed repository models pools, collateral, funding, trader behavior, and liquidations. The inspected model includes base-asset collateral handling. D8X's documentation describes inverse as well as linear and quanto products.

**Use:** evaluate simulation concepts and stress scenarios. Verify its contract-size, entry, and P&L conventions against our fixtures rather than assuming every product described as inverse is identical to Binance COIN-M. A complete deployable D8X exchange-core repository was not verified during this research.

### hyperliquid-dev/hyper-evm-lib

[Repository](https://github.com/hyperliquid-dev/hyper-evm-lib)

This MIT-licensed library provides precompile access, CoreWriter/transfer helpers, token resolution, and local HyperCore simulation. Its maintainers identify themselves as Obsidian Audits.

**Use:** inspect and pin integration utilities if needed. Maintenance by an audit team is not evidence that our market or every dependency revision is audited. This library does not implement inverse-market economics.

## Decisions already selected

| Decision | Reason |
| --- | --- |
| Custom HyperEVM contracts | Allows the required BTC-denominated inverse accounting |
| One BTC perpetual and fixed USD contracts | Keeps the first product close to the user's requested COIN-M behavior |
| MockBTC for the first testnet | Enables controlled testing without real BTC transfers or an assumed testnet bridge |
| Isolated, one-way positions | Provides a bounded first account-risk model |
| Pool counterparty | Implements the requested independent trading market with seeded test liquidity |
| Independent reference engine before contracts | Makes accounting discrepancies observable |
| Separate live and controlled price deployments | Supports both realistic operation and repeatable adverse scenarios |
| HyperCore hedging after baseline acceptance | Avoids making asynchronous hedge execution a prerequisite for the first market |

## Open dependencies and limits

- No selected repository has been established as a complete, production-ready HyperEVM BTC inverse exchange.
- Dependency revisions, source-level reuse decisions, and license obligations must be recorded before importing code.
- Testnet RPCs, faucet eligibility, BTC feed availability, and the explorer verification route have not been operationally tested.
- The first release will not need mainnet deposits or real BTC. If test resources are unavailable, use local infrastructure while recording the deployment blocker.
- A BTC pool may become insolvent after a sufficiently severe move or operational failure. The plan requires testing and defining the failure behavior; it does not promise that a finite reserve can pay every possible outcome.
- Exact parameters and execution/oracle semantics remain Stage 0 decisions. Their checklists are in [Market specification](MARKET_SPEC.md).

## Decision record format for implementation

```text
ID and date:
Question:
Selected behavior:
Alternatives considered:
Evidence/source revision:
Effect on trader accounting and pool risk:
Required tests:
Status and remaining uncertainty:
```
