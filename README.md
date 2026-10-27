# HyperEVM BTC Inverse Perpetual

Build a custom BTC/USD inverse perpetual market on HyperEVM testnet. Each trader deposits a test BTC token, independently opens a long or short, and settles margin, profit, loss, fees, and funding in that token.

## Motivation

This project explores how to build a BTC-margined inverse perpetual market on HyperEVM. In this model, contracts have a fixed USD value while collateral and trading outcomes are denominated in BTC. The goal is to make the market's mechanics transparent and testable, from inverse payoff and funding to liquidity-provider risk and liquidation, while creating a working testnet demonstration that developers can study and improve together.

The project is in its planning stage and welcomes contributors interested in smart contracts, derivatives modeling, oracle design, risk simulations, keeper infrastructure, and trading interfaces. See the [roadmap](docs/ROADMAP.md) for proposed work and open a GitHub issue to discuss an idea or coordinate a contribution. Reviews of the specifications and assumptions are valuable too.

The reference is Binance BTC COIN-M perpetuals: fixed USD contract value and inverse BTC payoff. A BTC liquidity pool supplies the counterparty capital. The first release implements Binance-like **one-way, isolated-margin trading** with deliberately limited exposure.

**Status:** planning documented on 2026-10-27. Implementation, simulation runs, security verification, and deployment have not started. External sources were reviewed on 2026-10-26/27; live parameters must be checked again during Stage 0.

## Documents

| Document | Purpose |
| --- | --- |
| [Staged roadmap](docs/ROADMAP.md) | Work packages, dependencies, deliverables, and gates |
| [Market specification](docs/MARKET_SPEC.md) | Contract economics, BTC accounting, and differences from Binance |
| [Architecture](docs/ARCHITECTURE.md) | Contracts, prices, pool, keepers, application, and boundaries |
| [Testnet acceptance](docs/TESTNET_ACCEPTANCE.md) | Evidence required to declare the objective achieved |
| [Research and sources](docs/RESEARCH.md) | Official references, GitHub candidates, and unresolved assumptions |

## Release target

- HyperEVM testnet, chain ID **998**, with HYPE for gas.
- One market, `BTCUSD-INVERSE`, with **$100 face value per contract**.
- An explicitly labeled, mintable **MockBTC** token with 8 decimals; it has no BTC backing or redemption promise.
- BTC-denominated margin, P&L, trading fees, funding, liquidation proceeds, and withdrawals.
- Separate trader accounts; one net position per account; isolated margin.
- Market requests, limit orders, partial closes, reduce-only orders, and basic stop-loss/take-profit requests.
- A BTC pool with LP shares, exposure limits, reserves, and a defined insolvency procedure.
- Live BTC/USD observations plus a separate controlled-price deployment for reproducible stress demonstrations.
- A usable testnet interface, keeper services, deployment records, and a completed 48-hour observation run.

The public demonstration must visibly distinguish mock collateral from real BTC. A controlled-price demonstration alone does not satisfy the live-price release gate.

## Work sequence

```mermaid
flowchart LR
    A[0. Freeze rules and check testnet] --> B[1. Reference model and pool simulation]
    B --> C[2. Contracts and accounting]
    C --> D[3. Prices, execution, and funding]
    D --> E[4. Liquidation and pool stress tests]
    E --> F[5. Keepers and trading interface]
    F --> G[6. Integrated rehearsal]
    G --> H[7. Deploy and demonstrate on testnet]
```

HyperCore hedging, real BTC token integration, cross margin, simultaneous long/short hedge mode, additional markets, and mainnet deployment are later extensions. The testnet market must first work with its own funded pool. Its completion criteria are in [Testnet acceptance](docs/TESTNET_ACCEPTANCE.md).

**First implementation task:** Stage 0. Resolve the oracle, funding, execution, and pool-risk decisions before writing the exchange contracts.
