# Murmur Dashboard — Grid Trader UI

## Overview

A real-time dashboard for the Murmur autonomous DeFi agent, styled as a Nasdaq-like trading terminal. Pure black background, 1px grid borders, no rounded corners, maximum data density. Connects to the loop via WebSocket for instant updates.

## Visual Style

**Grid Trader** — pure grid layout, monospace typography, green/amber/red accent colors on black. No cards, no curves, no whitespace wasted. Closest to a Bloomberg/trading terminal aesthetic.

## Architecture

The loop process runs a WebSocket server on a configurable port (default `3001`). The React dashboard connects and renders all 6 panels from pushed state.

```
Loop process (tsx src/loop/index.ts)
  ├── Cron cycle (every 15m)
  └── WS server (:3001)
       ├── Broadcasts phase changes in real-time
       ├── Broadcasts full state snapshot on connect
       └── Broadcasts after each cycle completes

Dashboard (vite dev server :5173, or built static)
  └── Connects to ws://localhost:3001
       └── Renders Grid Trader UI from WS state
```

## WebSocket Protocol

Server pushes JSON messages with a `type` field:

| Type | When | Payload |
|------|------|---------|
| `snapshot` | On connect + after each cycle | Full state: treasury, positions, scored assets, last N decisions, risk gate, cycle info |
| `phase` | Each phase transition | `{ cycleId, phase, timestamp }` |
| `trade` | After execution | `{ action, slug, amount, txHash }` |

### Snapshot payload shape

```typescript
interface DashboardSnapshot {
  treasury: TreasuryState;
  scoredAssets: ScoredAsset[];
  lastDecisions: DecisionLogEntry[];  // last 20
  riskGate: RiskGateResult | null;
  currentCycle: {
    cycleId: string;
    phase: LoopPhase;
    startedAt: string;
  } | null;
  config: {
    network: string;
    cronSchedule: string;
    dryRun: boolean;
    maxNotionalUsd: number;
    maxDailyTurnoverUsd: number;
  };
  cycleCount: number;
  uptimeSince: string;
}

interface DecisionLogEntry {
  cycleId: string;
  timestamp: string;
  action: TradeAction;
  slug: AssetSlug | null;
  thesis: string;
  confidence: number;
  riskApproved: boolean;
  effectiveSizeUsd: number;
  result: "executed" | "blocked" | "hold" | "dry-run";
  pnlPct: number | null;
}
```

## Dashboard Sections (6 panels)

### 1. Header Bar
- Left: "MURMUR" in bold monospace, letter-spacing
- Right: network name, cycle count, UTC time, countdown to next cycle

### 2. Cycle Progress Bar
- 6-phase pipeline: SENSE → NORM → SCORE → DELIB → RISK → EXEC
- Dots: green (done), amber pulsing (active), dark (pending)
- Cycle ID on the left

### 3. Summary Stats (6 columns)
- NAV (total portfolio USD)
- USDC (available balance)
- P&L 24H (percentage + trade count)
- CAP LEFT (delegation cap remaining)
- DAILY VOL (daily turnover spent vs limit)
- LAST ACTION (action + asset + time ago + size)

### 4. Open Positions Table
- Columns: ASSET, VALUE, RET (return %), SCORE (composite), TTL (holding horizon)
- Header shows count vs max (e.g., "3/5")

### 5. Signal Heatmap
- Rows: 7 assets (deduplicated, no duplicate ethereum/weth)
- Columns: 9 Santiment metrics + COMP (composite score)
- Cells color-coded: green (high positive), yellow (mid), red (negative), gray (neutral)
- Values show z-scores or normalized scores

### 6. Risk Gate (14 checks)
- 2-column grid of check names with colored dots (green pass, amber warn, red fail)
- Bottom summary: APPROVED/BLOCKED status, effective size, slippage cap, delegation remaining

### 7. Decision Log
- Last 20 cycles in reverse chronological order
- Columns: TIME, ACTION, ASSET, THESIS (truncated), RESULT (P&L % or status)

## Frontend Stack

- **Vite + React 19 + TypeScript** — SPA
- **Plain CSS** — custom terminal theme, no Tailwind (the aesthetic is too specific)
- **Monorepo** — shares root `package.json` and `src/types/`

## File Structure

```
dashboard/
  ├── index.html
  ├── vite.config.ts
  ├── src/
  │   ├── main.tsx
  │   ├── App.tsx              — Layout grid + WS connection
  │   ├── hooks/
  │   │   └── useSocket.ts     — WS hook with auto-reconnect
  │   ├── components/
  │   │   ├── Header.tsx
  │   │   ├── CycleBar.tsx
  │   │   ├── Summary.tsx
  │   │   ├── Positions.tsx
  │   │   ├── Heatmap.tsx
  │   │   ├── RiskGate.tsx
  │   │   └── DecisionLog.tsx
  │   └── styles/
  │       └── terminal.css     — Grid Trader theme
src/
  ├── ws/index.ts              — WS server module
  └── loop/index.ts            — Starts WS server alongside cron
```

## WS Server Module (`src/ws/index.ts`)

- Uses `ws` npm package
- Starts on configurable port (env `WS_PORT`, default `3001`)
- Maintains set of connected clients
- Exports `broadcast(type, payload)` function called by the loop
- Sends `snapshot` to each new client on connect
- No authentication (local-only for now)

## Loop Integration

The loop calls into the WS module at these points:
- `main()` startup: start WS server
- Each phase transition in `runCycle`: broadcast `phase` message
- After execution: broadcast `trade` message
- End of cycle: broadcast `snapshot` with full updated state

## New Dependencies

- `ws` — WebSocket server
- `@types/ws` — TypeScript types
- `react`, `react-dom`, `@types/react`, `@types/react-dom` — React
- `vite`, `@vitejs/plugin-react` — Build tooling
