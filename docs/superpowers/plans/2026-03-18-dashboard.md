# Murmur Dashboard Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a real-time Grid Trader dashboard that connects to the Murmur loop via WebSocket and renders portfolio, signals, risk gate, and decision log in a Nasdaq-style terminal UI.

**Architecture:** The loop process starts a WS server alongside the cron. It broadcasts phase transitions and full state snapshots. A Vite+React SPA in `dashboard/` connects via WebSocket and renders 7 panels (header, cycle bar, summary, positions, heatmap, risk gate, decision log) in a pure-black grid layout.

**Tech Stack:** TypeScript, ws, React 19, Vite, plain CSS

---

## File Structure

### New files:
- `src/types/dashboard.ts` — Dashboard-specific types (snapshot, decision log entry, WS messages)
- `src/ws/index.ts` — WebSocket server module
- `dashboard/index.html` — Vite HTML entry
- `dashboard/vite.config.ts` — Vite config
- `dashboard/tsconfig.json` — Dashboard-specific TypeScript config
- `dashboard/src/main.tsx` — React mount
- `dashboard/src/App.tsx` — Layout grid + WS connection
- `dashboard/src/hooks/useSocket.ts` — WS hook with auto-reconnect
- `dashboard/src/components/Header.tsx`
- `dashboard/src/components/CycleBar.tsx`
- `dashboard/src/components/Summary.tsx`
- `dashboard/src/components/Positions.tsx`
- `dashboard/src/components/Heatmap.tsx`
- `dashboard/src/components/RiskGate.tsx`
- `dashboard/src/components/DecisionLog.tsx`
- `dashboard/src/styles/terminal.css` — Grid Trader theme

### Modified files:
- `package.json` — Add dependencies and scripts
- `src/loop/index.ts` — Import WS module, broadcast phase changes and snapshots
- `tsconfig.json` — Exclude `dashboard/` from backend tsconfig

---

### Task 1: Install Dependencies and Configure Build

**Files:**
- Modify: `package.json`
- Modify: `tsconfig.json`
- Create: `dashboard/tsconfig.json`
- Create: `dashboard/vite.config.ts`
- Create: `dashboard/index.html`

- [ ] **Step 1: Add dependencies to package.json**

Add to `dependencies`:
```json
"ws": "^8.18.0"
```

Add to `devDependencies`:
```json
"@types/ws": "^8.5.13",
"react": "^19.0.0",
"react-dom": "^19.0.0",
"@types/react": "^19.0.0",
"@types/react-dom": "^19.0.0",
"vite": "^6.0.0",
"@vitejs/plugin-react": "^4.3.0"
```

Add scripts:
```json
"dashboard": "vite --config dashboard/vite.config.ts",
"dashboard:build": "vite build --config dashboard/vite.config.ts"
```

- [ ] **Step 2: Run npm install**

```bash
npm install
```

- [ ] **Step 3: Exclude dashboard/ from backend tsconfig**

In `tsconfig.json`, add `"dashboard"` to the exclude array:

```json
"exclude": ["node_modules", "dist", "dashboard"]
```

- [ ] **Step 4: Create dashboard/tsconfig.json**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "jsx": "react-jsx",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "noUnusedLocals": false,
    "noUnusedParameters": false
  },
  "include": ["src/**/*", "../src/types/**/*"]
}
```

- [ ] **Step 5: Create dashboard/vite.config.ts**

```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import path from "path";

export default defineConfig({
  plugins: [react()],
  root: path.resolve(__dirname),
  resolve: {
    alias: {
      "@types": path.resolve(__dirname, "../src/types"),
    },
  },
  server: {
    port: 5173,
  },
});
```

- [ ] **Step 6: Create dashboard/index.html**

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>MURMUR</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

- [ ] **Step 7: Verify build passes**

```bash
npm run build
```

- [ ] **Step 8: Commit**

```bash
git add package.json package-lock.json tsconfig.json dashboard/
git commit -m "chore: add dashboard dependencies, Vite config, and TypeScript setup"
```

---

### Task 2: Dashboard Types and WS Server Module

**Files:**
- Create: `src/types/dashboard.ts`
- Modify: `src/types/index.ts` — re-export dashboard types
- Create: `src/ws/index.ts`

- [ ] **Step 1: Create dashboard types**

Create `src/types/dashboard.ts`:

```typescript
import type {
  TreasuryState,
  ScoredAsset,
  RiskGateResult,
  LoopPhase,
  TradeAction,
  AssetSlug,
} from "./index.js";

export interface DecisionLogEntry {
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

export interface DashboardSnapshot {
  treasury: TreasuryState;
  scoredAssets: ScoredAsset[];
  lastDecisions: DecisionLogEntry[];
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

export type WsMessage =
  | { type: "snapshot"; data: DashboardSnapshot }
  | { type: "phase"; data: { cycleId: string; phase: LoopPhase; timestamp: string } }
  | { type: "trade"; data: { action: TradeAction; slug: AssetSlug; amount: number; txHash: string } };
```

- [ ] **Step 2: Re-export from types/index.ts**

Add at the bottom of `src/types/index.ts`:

```typescript
export type {
  DecisionLogEntry,
  DashboardSnapshot,
  WsMessage,
} from "./dashboard.js";
```

- [ ] **Step 3: Create WS server module**

Create `src/ws/index.ts`:

```typescript
import { WebSocketServer, WebSocket } from "ws";
import type { DashboardSnapshot, WsMessage } from "../types/index.js";

let wss: WebSocketServer | null = null;
let latestSnapshot: DashboardSnapshot | null = null;

export function startWsServer(port: number = 3001): WebSocketServer {
  wss = new WebSocketServer({ port });

  wss.on("connection", (ws) => {
    console.log(`[WS] Client connected (${wss!.clients.size} total)`);

    // Send latest snapshot on connect
    if (latestSnapshot) {
      send(ws, { type: "snapshot", data: latestSnapshot });
    }

    ws.on("close", () => {
      console.log(`[WS] Client disconnected (${wss!.clients.size} total)`);
    });
  });

  console.log(`[WS] Server listening on ws://localhost:${port}`);
  return wss;
}

export function broadcast(message: WsMessage): void {
  if (!wss) return;

  // Cache snapshots for new clients
  if (message.type === "snapshot") {
    latestSnapshot = message.data;
  }

  const payload = JSON.stringify(message, (_key, value) =>
    typeof value === "bigint" ? value.toString() : value,
  );

  for (const client of wss.clients) {
    if (client.readyState === WebSocket.OPEN) {
      client.send(payload);
    }
  }
}

export function broadcastPhase(cycleId: string, phase: string, timestamp: string): void {
  broadcast({
    type: "phase",
    data: { cycleId, phase: phase as DashboardSnapshot["currentCycle"] extends null ? never : NonNullable<DashboardSnapshot["currentCycle"]>["phase"], timestamp },
  });
}

export function broadcastSnapshot(snapshot: DashboardSnapshot): void {
  broadcast({ type: "snapshot", data: snapshot });
}

export function stopWsServer(): void {
  if (wss) {
    wss.close();
    wss = null;
    console.log("[WS] Server stopped");
  }
}

function send(ws: WebSocket, message: WsMessage): void {
  ws.send(JSON.stringify(message, (_key, value) =>
    typeof value === "bigint" ? value.toString() : value,
  ));
}
```

- [ ] **Step 4: Verify build passes**

```bash
npm run build
```

- [ ] **Step 5: Commit**

```bash
git add src/types/dashboard.ts src/types/index.ts src/ws/index.ts
git commit -m "feat: add dashboard types and WebSocket server module"
```

---

### Task 3: Integrate WS into the Loop

**Files:**
- Modify: `src/loop/index.ts`

- [ ] **Step 1: Import WS module and dashboard types**

Add to the imports at the top of `src/loop/index.ts`:

```typescript
import { startWsServer, broadcastPhase, broadcastSnapshot } from "../ws/index.js";
import type { DashboardSnapshot, DecisionLogEntry } from "../types/index.js";
```

- [ ] **Step 2: Add dashboard state variables**

After the existing state variables (around line 94), add:

```typescript
let decisionLog: DecisionLogEntry[] = [];
let lastScoredAssets: ScoredAsset[] = [];
let lastRiskGate: import("../types/index.js").RiskGateResult | null = null;
let cycleCount = 0;
const uptimeSince = new Date().toISOString();
```

- [ ] **Step 3: Add buildSnapshot helper**

Add after the state variables:

```typescript
function buildSnapshot(
  config: ReturnType<typeof loadConfig>,
  currentCycle: DashboardSnapshot["currentCycle"] = null,
): DashboardSnapshot {
  return {
    treasury: treasuryState ?? {
      usdcBalance: BigInt(0),
      totalPortfolioUsd: 0,
      positions: [],
      lastUpdatedAt: new Date().toISOString(),
    },
    scoredAssets: lastScoredAssets,
    lastDecisions: decisionLog.slice(-20),
    riskGate: lastRiskGate,
    currentCycle,
    config: {
      network: "Base",
      cronSchedule: config.cronSchedule,
      dryRun: config.dryRun,
      maxNotionalUsd: config.delegationMaxNotionalUsd,
      maxDailyTurnoverUsd: config.delegationDailyTurnoverUsd,
    },
    cycleCount,
    uptimeSince,
  };
}
```

- [ ] **Step 4: Add phase broadcast calls in runCycle**

In the `runCycle` function, after each `cycle.phase = "..."` line, add a `broadcastPhase` call. Also store scored assets and risk gate results. The key insertions (add after each existing phase assignment):

After `cycle.phase = "sense"` (line ~420):
```typescript
    broadcastPhase(cycleId, "sense", new Date().toISOString());
```

After `cycle.phase = "normalize"` (line ~424):
```typescript
    broadcastPhase(cycleId, "normalize", new Date().toISOString());
```

After scoring completes (after `getTopCandidates`, around line ~428):
```typescript
    lastScoredAssets = scored;
    broadcastPhase(cycleId, "score", new Date().toISOString());
```

After `cycle.phase = "deliberate"` (line ~435):
```typescript
    broadcastPhase(cycleId, "deliberate", new Date().toISOString());
```

After `cycle.phase = "risk_gate"` (line ~468):
```typescript
    broadcastPhase(cycleId, "risk_gate", new Date().toISOString());
```

After `runRiskGatePhase` call (line ~469):
```typescript
    lastRiskGate = riskGateResult;
```

After `cycle.phase = "execute"` (line ~476):
```typescript
    broadcastPhase(cycleId, "execute", new Date().toISOString());
```

- [ ] **Step 5: Add decision log entry and snapshot broadcast at end of cycle**

Before `return cycle;` in the success path (around line ~512), add:

```typescript
    // ── Dashboard: log decision and broadcast snapshot
    cycleCount++;
    const logEntry: DecisionLogEntry = {
      cycleId,
      timestamp: new Date().toISOString(),
      action: finalDeliberation.decision.action,
      slug: finalDeliberation.decision.action === "hold" ? null : finalDeliberation.decision.slug,
      thesis: finalDeliberation.decision.thesis,
      confidence: finalDeliberation.decision.confidence,
      riskApproved: riskGateResult.approved,
      effectiveSizeUsd: riskGateResult.effectiveSizeUsd,
      result: executionResult ? "executed" : riskGateResult.approved ? (config.dryRun ? "dry-run" : "hold") : "blocked",
      pnlPct: null,
    };
    decisionLog.push(logEntry);
    if (decisionLog.length > 50) decisionLog = decisionLog.slice(-50);

    broadcastSnapshot(buildSnapshot(config));
```

- [ ] **Step 6: Start WS server in main()**

In the `main()` function, after `registerSignalHandlers()` (around line ~620), add:

```typescript
  // Start WebSocket server for dashboard
  const wsPort = Number(process.env.WS_PORT ?? 3001);
  startWsServer(wsPort);
```

- [ ] **Step 7: Verify build passes**

```bash
npm run build
```

- [ ] **Step 8: Commit**

```bash
git add src/loop/index.ts
git commit -m "feat: integrate WebSocket broadcasts into loop — phase changes and snapshots"
```

---

### Task 4: Terminal CSS Theme

**Files:**
- Create: `dashboard/src/styles/terminal.css`

- [ ] **Step 1: Create the Grid Trader theme**

Create `dashboard/src/styles/terminal.css` with the full terminal theme. This CSS defines:
- Black background, monospace font stack
- Grid layout for the main dashboard
- Summary stats row with 1px grid borders
- Table styles for positions and heatmap
- Phase progress bar with pulse animation
- Check dots for risk gate
- Decision log entry grid
- Color classes: `.green`, `.red`, `.amber`, `.blue`
- Heatmap cell colors: `.heat-high`, `.heat-mid`, `.heat-low`, `.heat-neutral`

The CSS should match the Grid Trader mockup that was approved (option C from the brainstorm). Key properties:
- `background: #000`, `color: #e0e0e0`
- `font-family: 'SF Mono', 'Fira Code', 'Cascadia Code', monospace`
- `font-size: 11px`, `line-height: 1.5`
- Grid gaps rendered as 1px `#222` borders using `gap:1px; background:#222` on parent with `background:#000` on children
- No border-radius anywhere
- Stat labels: `color:#555; font-size:8px; letter-spacing:1px; text-transform:uppercase`
- Panel titles: same as stat labels with `border-bottom:1px solid #1a1a1a`

Refer to the mockup at `.superpowers/brainstorm/897119-1773869369/layout-grid.html` for exact CSS values.

- [ ] **Step 2: Commit**

```bash
git add dashboard/src/styles/terminal.css
git commit -m "feat: add Grid Trader terminal CSS theme"
```

---

### Task 5: useSocket Hook

**Files:**
- Create: `dashboard/src/hooks/useSocket.ts`

- [ ] **Step 1: Create the WebSocket hook**

```typescript
import { useState, useEffect, useRef, useCallback } from "react";
import type { DashboardSnapshot, WsMessage } from "@types/dashboard";

interface SocketState {
  snapshot: DashboardSnapshot | null;
  connected: boolean;
  currentPhase: { cycleId: string; phase: string; timestamp: string } | null;
}

export function useSocket(url: string): SocketState {
  const [snapshot, setSnapshot] = useState<DashboardSnapshot | null>(null);
  const [connected, setConnected] = useState(false);
  const [currentPhase, setCurrentPhase] = useState<SocketState["currentPhase"]>(null);
  const wsRef = useRef<WebSocket | null>(null);
  const reconnectTimer = useRef<ReturnType<typeof setTimeout>>();

  const connect = useCallback(() => {
    const ws = new WebSocket(url);
    wsRef.current = ws;

    ws.onopen = () => setConnected(true);

    ws.onclose = () => {
      setConnected(false);
      reconnectTimer.current = setTimeout(connect, 3000);
    };

    ws.onerror = () => ws.close();

    ws.onmessage = (event) => {
      try {
        const msg = JSON.parse(event.data as string) as WsMessage;

        if (msg.type === "snapshot") {
          setSnapshot(msg.data);
          setCurrentPhase(msg.data.currentCycle ? {
            cycleId: msg.data.currentCycle.cycleId,
            phase: msg.data.currentCycle.phase,
            timestamp: msg.data.currentCycle.startedAt,
          } : null);
        } else if (msg.type === "phase") {
          setCurrentPhase(msg.data);
          // Update snapshot's currentCycle in place
          setSnapshot((prev) =>
            prev ? { ...prev, currentCycle: { cycleId: msg.data.cycleId, phase: msg.data.phase as DashboardSnapshot["currentCycle"] extends null ? never : NonNullable<DashboardSnapshot["currentCycle"]>["phase"], startedAt: msg.data.timestamp } } : prev,
          );
        }
      } catch {
        // ignore parse errors
      }
    };
  }, [url]);

  useEffect(() => {
    connect();
    return () => {
      clearTimeout(reconnectTimer.current);
      wsRef.current?.close();
    };
  }, [connect]);

  return { snapshot, connected, currentPhase };
}
```

- [ ] **Step 2: Commit**

```bash
git add dashboard/src/hooks/useSocket.ts
git commit -m "feat: add useSocket hook with auto-reconnect"
```

---

### Task 6: Dashboard Components — Header, CycleBar, Summary

**Files:**
- Create: `dashboard/src/components/Header.tsx`
- Create: `dashboard/src/components/CycleBar.tsx`
- Create: `dashboard/src/components/Summary.tsx`

- [ ] **Step 1: Create Header component**

Shows MURMUR branding, network, cycle count, UTC time. The time updates every second via `setInterval`.

- [ ] **Step 2: Create CycleBar component**

6-phase pipeline: SENSE → NORM → SCORE → DELIB → RISK → EXEC. Receives `currentPhase` prop. Maps the `LoopPhase` type to display. Dots are green (done), amber pulsing (active), dark (pending).

- [ ] **Step 3: Create Summary component**

6 stats in a grid row: NAV, USDC, P&L 24H, CAP LEFT, DAILY VOL, LAST ACTION. Receives `snapshot` prop. Formats bigint USDC balance as `Number(balance) / 1e6`. Colors: green for positive P&L, red for negative.

- [ ] **Step 4: Commit**

```bash
git add dashboard/src/components/Header.tsx dashboard/src/components/CycleBar.tsx dashboard/src/components/Summary.tsx
git commit -m "feat: add Header, CycleBar, and Summary dashboard components"
```

---

### Task 7: Dashboard Components — Positions, Heatmap

**Files:**
- Create: `dashboard/src/components/Positions.tsx`
- Create: `dashboard/src/components/Heatmap.tsx`

- [ ] **Step 1: Create Positions component**

Table with columns: ASSET, VALUE, RET, SCORE, TTL. Receives `positions` from snapshot treasury. Shows "N/5" in panel title.

- [ ] **Step 2: Create Heatmap component**

Grid of 7 assets x 10 columns (9 metrics + COMP). Receives `scoredAssets` from snapshot. For each asset, reads `normalizedSignals.metrics` and shows `zScore` values. Cell color class based on value: `>0.5` → `heat-high`, `>0.2` → `heat-mid`, `<-0.2` → `heat-low`, else `heat-neutral`. Deduplicate ethereum/weth (skip "weth" slug). Shorten metric names for column headers (e.g. `social_dominance_total` → `SOC DOM`).

- [ ] **Step 3: Commit**

```bash
git add dashboard/src/components/Positions.tsx dashboard/src/components/Heatmap.tsx
git commit -m "feat: add Positions table and Signal Heatmap components"
```

---

### Task 8: Dashboard Components — RiskGate, DecisionLog

**Files:**
- Create: `dashboard/src/components/RiskGate.tsx`
- Create: `dashboard/src/components/DecisionLog.tsx`

- [ ] **Step 1: Create RiskGate component**

2-column grid of 14 check names with colored dots. Receives `riskGate` from snapshot. Maps check status to dot color class (`pass` → green, `warn` → amber, `fail` → red). Bottom summary shows approved/blocked, effective size, slippage, delegation remaining.

- [ ] **Step 2: Create DecisionLog component**

List of last 20 decisions in reverse chronological order. Columns: TIME (HH:MM), ACTION (colored), ASSET, THESIS (truncated with text-overflow:ellipsis), RESULT (P&L % or status text). Action colors: buy → amber, exit → green, hold → gray, blocked → red.

- [ ] **Step 3: Commit**

```bash
git add dashboard/src/components/RiskGate.tsx dashboard/src/components/DecisionLog.tsx
git commit -m "feat: add RiskGate checks and DecisionLog components"
```

---

### Task 9: App Layout and Entry Point

**Files:**
- Create: `dashboard/src/main.tsx`
- Create: `dashboard/src/App.tsx`

- [ ] **Step 1: Create main.tsx**

```typescript
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import { App } from "./App";
import "./styles/terminal.css";

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <App />
  </StrictMode>,
);
```

- [ ] **Step 2: Create App.tsx**

Main layout using CSS grid. Connects to WS via `useSocket("ws://localhost:3001")`. Renders all 7 components in the Grid Trader layout:

```
Header (full width)
CycleBar (full width)
Summary (full width, 6 columns)
Positions (1/3) | Heatmap (2/3)
RiskGate (1/3) | DecisionLog (2/3)
```

Shows a "Connecting..." state when `!connected`. Shows "Waiting for data..." when `connected && !snapshot`.

- [ ] **Step 3: Verify dashboard starts**

```bash
npm run dashboard
```

Open http://localhost:5173 — should show "Connecting..." (loop isn't running).

- [ ] **Step 4: Commit**

```bash
git add dashboard/src/main.tsx dashboard/src/App.tsx
git commit -m "feat: add App layout and React entry point — dashboard complete"
```

---

### Task 10: End-to-End Verification

- [ ] **Step 1: Start the loop in dry-run mode**

```bash
DRY_RUN=true npm start
```

Verify the terminal shows `[WS] Server listening on ws://localhost:3001`.

- [ ] **Step 2: Start the dashboard in a second terminal**

```bash
npm run dashboard
```

Open http://localhost:5173. Verify:
- Header shows "MURMUR", network, cycle count
- Cycle bar animates through phases as the loop runs
- Summary stats populate from treasury state
- Signal heatmap fills with scored asset data
- Risk gate shows 14 checks
- Decision log shows the cycle result

- [ ] **Step 3: Commit any fixes**

```bash
git add -A
git commit -m "fix: dashboard end-to-end verification fixes"
```
