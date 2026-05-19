# sdl-indexer

> Soroban event processor and protocol decoders for Stellar DeFi Lens.

<p>
  <a href="https://www.drips.network/wave/stellar"><img src="https://img.shields.io/badge/Stellar-Wave%20Program-blueviolet?style=flat-square" /></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-MIT-22c55e?style=flat-square" /></a>
  <img src="https://img.shields.io/badge/Node.js-20%2B-339933?style=flat-square" />
  <img src="https://img.shields.io/badge/TypeScript-5.x-3178C6?style=flat-square" />
</p>

Part of [Stellar DeFi Lens](https://github.com/stellar-defi-lens) — the open-source DeFi analytics layer for Stellar.

---

## Overview

`sdl-indexer` is the data backbone of Stellar DeFi Lens. It connects to Soroban RPC, continuously polls contract events from every tracked DeFi protocol, decodes raw `ScVal` XDR into structured TypeScript objects, and writes clean time-series metrics into PostgreSQL.

Every other service in the project — `sdl-api`, `sdl-web`, `sdl-sdk` — depends on the data this package produces.

---

## How it works

```
Soroban RPC
    │
    │  getEvents() — polled every 6 seconds per contract
    ▼
EventPoller
    │
    ├──▶ BlendDecoder       supply · withdraw · borrow · repay · bad_debt
    ├──▶ AquariusDecoder    swap · add_liquidity · remove_liquidity
    ├──▶ SoroswapDecoder    swap (Router) · pair volume
    └──▶ TemplarDecoder     vault_deposit · vault_borrow · vault_repay
              │
              ▼
       PostgreSQL store
  (ledger-keyed · idempotent · time-series tables)
```

Each **decoder** is a class that knows:
1. Which contract IDs to watch
2. What event names that contract emits
3. How to decode each event's `topic` and `value` ScVal fields into typed metrics

Decoded events are written with the Stellar ledger sequence number as the primary key — this makes all writes **idempotent**, so re-indexing or backfilling is always safe.

---

## Project structure

```
sdl-indexer/
├── src/
│   ├── rpc/
│   │   ├── client.ts            # Soroban RPC wrapper (getEvents, getLedger)
│   │   └── poller.ts            # Per-contract polling loop with retry + backoff
│   ├── decoders/
│   │   ├── base.decoder.ts      # Abstract BaseDecoder — all decoders extend this
│   │   ├── blend.decoder.ts     # Blend Protocol: supply / borrow / repay / bad_debt
│   │   ├── aquarius.decoder.ts  # Aquarius AMM: swap / add_liquidity / remove_liquidity
│   │   ├── soroswap.decoder.ts  # Soroswap Router: swap events + pair volume
│   │   └── templar.decoder.ts   # Templar Protocol: vault deposit / borrow / repay
│   ├── store/
│   │   ├── db.ts                # pg connection pool (shared singleton)
│   │   ├── tvl.store.ts         # TVL snapshot upserts
│   │   ├── volume.store.ts      # Swap / volume event inserts
│   │   ├── rates.store.ts       # Borrow / supply rate snapshots (Blend)
│   │   └── events.store.ts      # Raw decoded event log (all protocols)
│   ├── types/
│   │   └── index.ts             # DecodedEvent, ProtocolId, TVLSnapshot, etc.
│   └── index.ts                 # Entry point — registers decoders, starts pollers
├── db/
│   └── schema.sql               # Full PostgreSQL schema + indexes + materialized views
├── .env.example
├── docker-compose.yml           # Postgres for local development
├── package.json
├── tsconfig.json
└── CONTRIBUTING.md
```

---

## Getting started

### Prerequisites

- **Node.js 20+**
- **pnpm** (`npm install -g pnpm`)
- **Docker** — for local Postgres
- **Stellar Mainnet RPC URL** — free options:
  - [OBSRVR](https://obsrvr.stellar.org/)
  - [ValidationCloud](https://validationcloud.io/)
  - [QuickNode](https://quicknode.com/)

### Setup

```bash
# 1. Clone the repo
git clone https://github.com/stellar-defi-lens/sdl-indexer.git
cd sdl-indexer

# 2. Install dependencies
pnpm install

# 3. Configure environment
cp .env.example .env
# Open .env and fill in RPC_URL and DATABASE_URL

# 4. Start Postgres
docker-compose up -d

# 5. Run database migrations
pnpm db:migrate

# 6. Start the indexer
pnpm dev
```

You should see structured log output confirming each poller has started and events are flowing:

```
{"level":"info","msg":"BlendDecoder registered","contracts":["CBLEND...","CBLEND2..."]}
{"level":"info","msg":"AquariusDecoder registered","contracts":["CAQUA..."]}
{"level":"info","msg":"Pollers started","count":4}
{"level":"info","msg":"Event decoded","protocol":"blend","type":"supply","ledger":54823901}
```

### Available scripts

```bash
pnpm dev          # Start with hot reload (tsx watch)
pnpm build        # Compile TypeScript to dist/
pnpm start        # Run compiled output (production)
pnpm test         # Run all unit tests (Vitest)
pnpm test:watch   # Watch mode
pnpm test:cov     # Coverage report
pnpm db:migrate   # Apply schema.sql to the database
pnpm db:reset     # Drop and re-migrate (destructive — dev only)
pnpm lint         # ESLint
pnpm typecheck    # tsc --noEmit
```

### Environment variables

```env
# ── Required ─────────────────────────────────────────────────────────
RPC_URL=https://mainnet.stellar.validationcloud.io/v1/YOUR_KEY
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/sdl

# ── Optional ─────────────────────────────────────────────────────────
NETWORK=mainnet               # mainnet | testnet (default: mainnet)
POLL_INTERVAL_MS=6000         # How often to poll each contract (default: 6000)
LOG_LEVEL=info                # debug | info | warn | error (default: info)
START_LEDGER=                 # Ledger to start from — empty means latest tip
```

---

## Protocol contract IDs (mainnet)

| Protocol | Contract / Reference |
|----------|---------------------|
| Blend Pool Factory | [blend-capital/blend-utils](https://github.com/blend-capital/blend-utils) — pools listed in `deployments/mainnet.json` |
| Aquarius AMM | [AquaToken/soroban-amm](https://github.com/AquaToken/soroban-amm) |
| Soroswap Factory | `CA4HEQTL2WPEUYKYKCDOHCDNIV4QHNJ7EL4J4NQ6VADP7SYHVRYZ7AW2` |
| Templar Protocol | See Templar documentation |
| Reflector Oracle | [reflector-network/reflector-contract](https://github.com/reflector-network/reflector-contract) |

---

## Decoder anatomy

Every decoder extends `BaseDecoder`:

```typescript
// src/decoders/base.decoder.ts
export abstract class BaseDecoder {
  /** Human-readable protocol identifier — used in DB writes and logs */
  abstract protocolId: ProtocolId;

  /** Contract IDs this decoder listens to */
  abstract contractIds: string[];

  /**
   * Decode a raw Soroban ContractEvent into a typed DecodedEvent.
   * Return null if the event should be skipped (wrong type, parse error, etc.)
   */
  abstract decode(event: ContractEvent, ledger: number): DecodedEvent | null;
}
```

A Soroban `ContractEvent` contains:
- `contractId` — the emitting contract address
- `topic` — array of `ScVal` (event name + parameters)
- `value` — event body as `ScVal`

Decode them with `scValToNative` from `@stellar/stellar-sdk`:

```typescript
import { scValToNative } from '@stellar/stellar-sdk';
import { BaseDecoder, ContractEvent, DecodedEvent, ProtocolId } from '../types';

export class BlendDecoder extends BaseDecoder {
  protocolId: ProtocolId = 'blend';
  contractIds = ['CBLEND1...', 'CBLEND2...'];

  decode(event: ContractEvent, ledger: number): DecodedEvent | null {
    const [eventName, assetAddr, userAddr] = event.topic.map(scValToNative);
    const amount = scValToNative(event.value);

    if (eventName !== 'supply') return null;

    return {
      protocol: this.protocolId,
      type: 'supply',
      asset: assetAddr as string,
      user: userAddr as string,
      amount: BigInt(amount),
      ledger,
      timestamp: Date.now(),
    };
  }
}
```

---

## Database schema overview

| Table | Populated by | Description |
|-------|-------------|-------------|
| `decoded_events` | All decoders | Raw event log — every decoded event, all protocols |
| `tvl_snapshots` | All decoders | Hourly TVL per protocol + asset |
| `volume_events` | Aquarius, Soroswap, Phoenix | Per-swap volume records |
| `blend_pool_snapshots` | BlendDecoder | Per-pool borrow/supply rates + utilization |
| `liquidation_events` | BlendDecoder | Liquidation + bad_debt events |
| `pair_volume` | Soroswap, Aquarius | Aggregated 24h + cumulative volume per token pair |

Full schema with column definitions: [`db/schema.sql`](db/schema.sql)

---

## Key dependencies

| Package | Purpose |
|---------|---------|
| `@stellar/stellar-sdk` | Soroban RPC client + `scValToNative` XDR decoding |
| `@blend-capital/blend-sdk` | Typed Blend Protocol pool state reads |
| `pg` | PostgreSQL client |
| `zod` | Runtime validation of decoded event shapes |
| `pino` | Structured JSON logging |

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for setup, decoder guide, and PR process.

Issues are labeled for the **Stellar Wave Program**:
- `wave:trivial` — 100 pts — isolated fixes, tests, error handling
- `wave:medium` — 150 pts — new event type, new store method
- `wave:high` — 200 pts — new protocol decoder, schema change, architectural feature

---

## License

MIT © Stellar DeFi Lens contributors
