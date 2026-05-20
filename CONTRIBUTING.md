# Contributing to sdl-indexer

Welcome! `sdl-indexer` is part of the **[Stellar Wave Program](https://www.drips.network/wave/stellar)** on Drips. Contributors who resolve labeled issues during a Wave earn **XLM rewards** funded by the Stellar Development Foundation.

This guide is written against the real project structure. Read it before opening a PR.

---

## Table of contents

- [Project structure](#project-structure)
- [How the system works](#how-the-system-works)
- [Local setup](#local-setup)
- [Understanding decoders](#understanding-decoders)
- [How to add a new protocol decoder](#how-to-add-a-new-protocol-decoder)
- [How to add a new event type to an existing decoder](#how-to-add-a-new-event-type-to-an-existing-decoder)
- [Working with the event bus](#working-with-the-event-bus)
- [Working with workers](#working-with-workers)
- [Working with the store](#working-with-the-store)
- [Testing](#testing)
- [Pull request process](#pull-request-process)
- [Code style](#code-style)

---

## Project structure

```
sdl-indexer/
├── .github/                      # CI workflows and Wave issue templates
├── docker/                       # Dockerfile and Postgres setup files
├── scripts/                      # One-off scripts (backfill, seed, migration helpers)
├── src/
│   ├── bus/
│   │   └── events.ts             # Typed event emitter — all decoded events flow through here
│   ├── config/                   # Environment config loader and Zod validation
│   ├── decoders/
│   │   ├── aquarius/             # Aquarius AMM decoder (mirrors blend/ structure)
│   │   ├── blend/
│   │   │   ├── constants.ts      # Contract IDs and known event name strings
│   │   │   ├── decoder.ts        # Orchestrator — routes events to parser functions
│   │   │   ├── parser.ts         # Pure ScVal parsing functions, one per event type
│   │   │   └── types.ts          # TypeScript types for all Blend decoded events
│   │   └── soroswap/             # Soroswap Router decoder (mirrors blend/ structure)
│   ├── rpc/
│   │   ├── client.ts             # Soroban RPC wrapper (getEvents, getLedger)
│   │   └── poller.ts             # Per-contract polling loop with retry and backoff
│   ├── store/
│   │   ├── db.ts                 # pg connection pool singleton
│   │   ├── events.ts             # Decoded event log writes
│   │   └── snapshots.ts          # TVL snapshot and rate upserts
│   ├── types/                    # Shared TypeScript types across the whole project
│   ├── utils/                    # Shared utilities (logger, math helpers, formatters)
│   └── workers/
│       ├── indexer.ts            # Coordinates decoder → bus → store per incoming event
│       └── index.ts              # Entry point — boots config, registers workers, starts pollers
├── .env.example
├── .gitignore
├── CONTRIBUTING.md
├── docker-compose.yml            # Local Postgres
└── package.json
```

---

## How the system works

The indexer has four distinct layers that work in sequence:

```
Soroban RPC
    │
    │  getEvents() — polled by rpc/poller.ts per contract
    ▼
workers/indexer.ts           ← receives ContractEvents, routes to correct decoder
    │
    ├──▶ decoders/blend/decoder.ts    ← calls parser.ts, returns typed DecodedEvent
    ├──▶ decoders/aquarius/decoder.ts
    └──▶ decoders/soroswap/decoder.ts
              │
              ▼
        bus/events.ts                ← decoded events published to the internal event bus
              │
              ▼
        store/{events.ts, snapshots.ts}   ← idempotent writes to PostgreSQL
```

**Why the decoder splits into `decoder.ts` + `parser.ts`:**
`decoder.ts` orchestrates — it reads the event name and routes to the right function. `parser.ts` contains the raw `scValToNative` decoding logic, one function per event type. This separation keeps each file focused and makes individual parser functions independently testable without wiring up a full decoder.

**Why the event bus:**
Decoded events are published to `bus/events.ts` before being consumed by the store. This decoupling means new consumers (a WebSocket broadcaster, an alert system) can be added by subscribing to the bus — without touching decoder or store code.

---

## Local setup

### Requirements

- Node.js 20+
- pnpm (`npm install -g pnpm`)
- Docker and Docker Compose

### Steps

```bash
# 1. Fork and clone
git clone https://github.com/<your-username>/sdl-indexer.git
cd sdl-indexer

# 2. Install dependencies
pnpm install

# 3. Configure environment
cp .env.example .env
# Set RPC_URL — free options: ValidationCloud, OBSRVR, QuickNode
# DATABASE_URL defaults work with Docker as-is

# 4. Start Postgres
docker-compose up -d

# 5. Run migrations
pnpm db:migrate

# 6. Start the indexer
pnpm dev
```

Confirm it is running — you should see worker boot messages and decoded events appearing in the logs within a few seconds.

> **Tip:** Set `NETWORK=testnet` and use a testnet RPC URL to develop without mainnet costs. Most protocols have testnet deployments you can use to generate real fixture data.

### Available scripts

```bash
pnpm dev           # Start with hot reload (tsx watch)
pnpm build         # Compile TypeScript to dist/
pnpm start         # Run compiled output
pnpm test          # Run all tests (Vitest)
pnpm test:watch    # Watch mode
pnpm test:cov      # Coverage report
pnpm db:migrate    # Apply schema to connected database
pnpm db:reset      # Drop and re-migrate — destructive, dev only
pnpm lint          # ESLint
pnpm typecheck     # tsc --noEmit
```

### Environment variables

```env
# Required
RPC_URL=https://mainnet.stellar.validationcloud.io/v1/YOUR_KEY
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/sdl

# Optional
NETWORK=mainnet            # mainnet | testnet (default: mainnet)
POLL_INTERVAL_MS=6000      # Poll frequency per contract in ms (default: 6000)
LOG_LEVEL=info             # debug | info | warn | error (default: info)
START_LEDGER=              # Ledger to start from — empty = network tip
```

---

## Understanding decoders

Every decoder follows the same four-file structure. Use `src/decoders/blend/` as your reference.

### `constants.ts` — contract IDs and event names

```typescript
export const BLEND_CONTRACTS = {
  POOL_FACTORY: "CBLEND...",
  USDC_POOL: "CBLEND2...",
} as const;

export const BLEND_EVENTS = {
  SUPPLY: "supply",
  WITHDRAW: "withdraw",
  BORROW: "borrow",
  REPAY: "repay",
  BAD_DEBT: "bad_debt",
} as const;
```

No magic strings anywhere else in the codebase. All contract IDs and event names come from `constants.ts`.

### `types.ts` — decoded event types

Discriminated unions keyed on `type`. One interface per event type, one union that groups them all.

```typescript
export interface BlendSupplyEvent {
  protocol: "blend";
  type: "supply";
  poolAddress: string;
  asset: string;
  user: string;
  amount: bigint;
  ledger: number;
  timestamp: number;
}

export type BlendDecodedEvent =
  | BlendSupplyEvent
  | BlendWithdrawEvent
  | BlendBorrowEvent
  | BlendRepayEvent
  | BlendBadDebtEvent;
```

### `parser.ts` — pure ScVal parsing functions

One exported function per event type. Each function receives the raw `ContractEvent` and the ledger number, decodes using `scValToNative`, and returns the typed event object. No try/catch here — parsing errors bubble up to `decoder.ts`.

```typescript
import { scValToNative } from "@stellar/stellar-sdk";
import type { ContractEvent } from "../../types";
import type { BlendSupplyEvent } from "./types";

export function parseSupplyEvent(
  event: ContractEvent,
  ledger: number
): BlendSupplyEvent {
  const [, assetAddr, userAddr] = event.topic.map(scValToNative);
  const amount = BigInt(scValToNative(event.value));

  return {
    protocol: "blend",
    type: "supply",
    poolAddress: event.contractId,
    asset: assetAddr as string,
    user: userAddr as string,
    amount,
    ledger,
    timestamp: Date.now(),
  };
}
```

### `decoder.ts` — the orchestrator

Reads the event name from `topic[0]`, routes to the right parser, wraps in try/catch, and returns `null` on failure. Never throws.

```typescript
import { scValToNative } from "@stellar/stellar-sdk";
import { logger } from "../../utils";
import { BLEND_CONTRACTS, BLEND_EVENTS } from "./constants";
import { parseSupplyEvent, parseBorrowEvent } from "./parser";
import type { ContractEvent } from "../../types";
import type { BlendDecodedEvent } from "./types";

export class BlendDecoder {
  readonly contractIds = Object.values(BLEND_CONTRACTS);

  decode(event: ContractEvent, ledger: number): BlendDecodedEvent | null {
    const eventName = scValToNative(event.topic[0]) as string;

    try {
      switch (eventName) {
        case BLEND_EVENTS.SUPPLY:
          return parseSupplyEvent(event, ledger);
        case BLEND_EVENTS.BORROW:
          return parseBorrowEvent(event, ledger);
        default:
          return null;
      }
    } catch (err) {
      logger.warn({
        msg: "Decode failed",
        protocol: "blend",
        eventName,
        ledger,
        err,
      });
      return null;
    }
  }
}
```

---

## How to add a new protocol decoder

### Step 1 — Find the event spec

Look at the protocol's GitHub for `emit_event` or `publish_event` calls in the Rust contract source. You need:

- Mainnet contract address(es)
- Event names emitted
- Topic array structure (what each `ScVal` position represents)
- Value structure (what the event body contains)

Good sources: protocol GitHub repos, [Stellar Lab](https://lab.stellar.org) (search the contract address and inspect raw event logs).

### Step 2 — Create the decoder folder

```bash
mkdir src/decoders/<protocol>
touch src/decoders/<protocol>/constants.ts
touch src/decoders/<protocol>/types.ts
touch src/decoders/<protocol>/parser.ts
touch src/decoders/<protocol>/decoder.ts
touch src/decoders/<protocol>/parser.test.ts
touch src/decoders/<protocol>/decoder.test.ts
```

Copy `src/decoders/blend/` as your template. Keep the exact four-file structure.

### Step 3 — Register in the worker

Open `src/workers/index.ts`. Import your decoder and register it. The indexer worker in `src/workers/indexer.ts` will automatically start routing events from your decoder's `contractIds` to its `decode()` method.

### Step 4 — Add store writes if needed

If your decoder produces a new metric type:

- Add a write method in `src/store/snapshots.ts` or a new store file
- Consume your bus event in `src/workers/indexer.ts` and call the method
- Add any new tables to the schema (coordinate with a maintainer for schema changes)

### Step 5 — Write tests and open a PR

See [Testing](#testing). PR description must link to the protocol contract source showing the event spec you decoded.

---

## How to add a new event type to an existing decoder

1. Find the event spec in the protocol's Rust contract source
2. Add the constant to `constants.ts`
3. Add the TypeScript interface to `types.ts` and extend the union
4. Add a pure parser function to `parser.ts`
5. Add a `case` in `decoder.ts` that calls the new parser
6. Add test cases in both `parser.test.ts` and `decoder.test.ts`
7. Update the decoder's top-level JSDoc comment listing all supported event types

---

## Working with the event bus

`src/bus/events.ts` is a typed event emitter. All decoded events are published here before being written to the store.

If you add a new protocol, add its event channel names to the typed event map in `bus/events.ts` before emitting them. TypeScript will enforce this — if an event channel is not declared in the map, the compiler will catch it.

```typescript
// Adding a new protocol to the event bus type map
export interface EventMap {
  "blend:supply": BlendSupplyEvent;
  "blend:borrow": BlendBorrowEvent;
  // Add yours here:
  "phoenix:swap": PhoenixSwapEvent;
}
```

---

## Working with workers

`src/workers/indexer.ts` coordinates the flow between the RPC poller, the decoder registry, and the event bus. It receives raw `ContractEvent` objects, looks up the right decoder by `contractId`, calls `decode()`, and publishes the result to the bus.

`src/workers/index.ts` is the process entry point. It:

1. Loads and validates config from `src/config/`
2. Instantiates the RPC client from `src/rpc/client.ts`
3. Instantiates all decoder classes and registers them in the indexer worker
4. Starts the polling loop in `src/rpc/poller.ts`

You should only need to touch `workers/index.ts` when registering a new decoder or changing startup behavior.

---

## Working with the store

| File                 | Purpose                                                           |
| -------------------- | ----------------------------------------------------------------- |
| `store/db.ts`        | `pg` connection pool singleton — import `pool` from here          |
| `store/events.ts`    | Writes every decoded event to the raw `decoded_events` log table  |
| `store/snapshots.ts` | Upserts TVL snapshots, borrow/supply rates, and volume aggregates |

All writes must be **idempotent** — use `ON CONFLICT DO NOTHING` or `ON CONFLICT DO UPDATE` with a ledger-based conflict target. This guarantees that re-running or backfilling is always safe with no duplicates.

```typescript
// Correct — idempotent upsert
await pool.query(
  `
  INSERT INTO tvl_snapshots (protocol, asset, value_usd, ledger)
  VALUES ($1, $2, $3, $4)
  ON CONFLICT (protocol, asset, ledger) DO UPDATE
    SET value_usd = EXCLUDED.value_usd
`,
  [protocol, asset, valueUsd, ledger]
);
```

---

## Testing

```bash
pnpm test          # Run all tests
pnpm test:watch    # Watch mode
pnpm test:cov      # Coverage report
```

We use **Vitest**. Test files live next to the files they test (`parser.test.ts` beside `parser.ts`).

### Parser tests — use real fixture data

Copy a real `ContractEvent` JSON object from Stellar Lab or testnet. Paste it as a constant in the test file and assert the parser output.

```typescript
// src/decoders/blend/parser.test.ts
import { describe, it, expect } from 'vitest';
import { parseSupplyEvent } from './parser';

// Source: Stellar Lab, Blend USDC pool, tx hash 0xABC...
const FIXTURE_SUPPLY = { contractId: 'CBLEND2...', topic: [...], value: ... };

describe('parseSupplyEvent', () => {
  it('decodes a valid supply event', () => {
    const result = parseSupplyEvent(FIXTURE_SUPPLY, 54823901);
    expect(result.type).toBe('supply');
    expect(result.amount).toBeGreaterThan(0n);
    expect(typeof result.asset).toBe('string');
  });
});
```

### Decoder tests — cover the failure paths too

```typescript
// src/decoders/blend/decoder.test.ts
describe("BlendDecoder.decode", () => {
  const decoder = new BlendDecoder();

  it("returns a typed event for a valid supply", () => {
    expect(decoder.decode(FIXTURE_SUPPLY, 54823901)?.type).toBe("supply");
  });

  it("returns null for an unknown event name", () => {
    expect(decoder.decode(FIXTURE_UNKNOWN_EVENT, 54823901)).toBeNull();
  });

  it("does not throw on a malformed topic", () => {
    const bad = { ...FIXTURE_SUPPLY, topic: [] };
    expect(() => decoder.decode(bad, 54823901)).not.toThrow();
    expect(decoder.decode(bad, 54823901)).toBeNull();
  });
});
```

### Store tests — use pg-mem

Use `pg-mem` for an in-memory PostgreSQL instance in store tests. No Docker required for unit tests.

---

## Pull request process

1. **Branch from `main`**: `git checkout -b feat/phoenix-decoder`
2. **One PR per issue** — tight scope, easier review
3. **Checklist before opening:**
   - `pnpm test` passes
   - `pnpm typecheck` passes (no TypeScript errors)
   - `pnpm lint` passes
   - New decoder has all four files: `constants.ts`, `types.ts`, `parser.ts`, `decoder.ts`
   - Tests cover: valid event, unknown event name, malformed topic
4. **PR description must include:**
   - Link to the issue
   - Link to the protocol contract source showing the event spec
   - List of event types decoded
   - Any new store tables or schema changes

---

## Code style

- **TypeScript strict mode** — no `any`, no untyped assertions
- **Parser functions are pure** — no side effects, no logging, no try/catch
- **Decoders catch and return null** — never let parse errors propagate to the worker
- **Pino structured logging** — use the `logger` utility from `src/utils/`; no `console.log` in production code
- **Idempotent writes** — always use conflict clauses in all SQL inserts
- **Constants over magic strings** — every contract ID and event name lives in `constants.ts`

---

## Getting help

- Open a [GitHub Discussion](https://github.com/stellar-defi-lens/sdl-indexer/discussions)
- Join the [Stellar Developer Discord](https://discord.gg/stellar) in `#soroban`
- Tag questions with `sdl-indexer` so maintainers see them
