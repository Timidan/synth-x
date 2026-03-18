# Sponsor Integration Cleanup — Removals, Replacements & Fixes

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace Bankr with Venice AI for free LLM inference, remove false sponsor claims (bond.credit), enable Filecoin storage, re-enable fast-lane exits, and update the README to accurately reflect integrations.

**Architecture:** Swap the LLM gateway from Bankr (`X-API-Key` auth, `https://llm.bankr.bot`) to Venice AI (`Bearer` auth, `https://api.venice.ai/api/v1`). Venice is OpenAI-compatible so the request/response format stays identical — only the URL, auth header, model names, and env var change. Remove bond.credit track claim. Enable Filecoin uploads. Fix fast-lane exits.

**Tech Stack:** TypeScript, Venice AI API, Lighthouse/Filecoin

---

## File Structure

### Modified files:
- `src/strategist/index.ts` — Replace Bankr with Venice AI (URL, auth header, model names)
- `src/loop/index.ts` — Update env var references, re-enable fast-lane, remove Bankr mentions from startup log
- `src/types/index.ts` — Rename `bankrApiKey`/`bankrBaseUrl` to `veniceApiKey`/`veniceBaseUrl` in MurmurConfig
- `.env` — Replace `BANKR_API_KEY`/`BANKR_BASE_URL` with `VENICE_API_KEY`
- `README.md` — Remove bond.credit track, replace Bankr references with Venice AI, update prize tracks table
- `dashboard/src/components/Header.tsx` — No changes needed (reads from snapshot)

---

### Task 1: Replace Bankr with Venice AI in Strategist

**Files:**
- Modify: `src/strategist/index.ts`

- [ ] **Step 1: Update constants**

Replace the Bankr constants (around lines 30-35):

```typescript
// Old:
const BANKR_BASE_URL = process.env.BANKR_BASE_URL ?? "https://llm.bankr.bot";
const ROUTINE_MODEL = "claude-sonnet-4-6";
const HIGH_STAKES_MODEL = "claude-opus-4-5";

// New:
const VENICE_BASE_URL = process.env.VENICE_BASE_URL ?? "https://api.venice.ai/api/v1";
const ROUTINE_MODEL = "zai-org-glm-4.7";
const HIGH_STAKES_MODEL = "zai-org-glm-4.7"; // Venice uses same flagship model
```

- [ ] **Step 2: Rename types and function**

Rename `BankrMessage`, `BankrRequest`, `BankrChoice`, `BankrResponse` to `LLMMessage`, `LLMRequest`, `LLMChoice`, `LLMResponse` (these are generic OpenAI-compatible types, not Bankr-specific).

Rename `callBankr` → `callLLM`, `callBankrWithRetry` → `callLLMWithRetry`.

- [ ] **Step 3: Update the callLLM function**

Change the fetch call:
- URL: `${VENICE_BASE_URL}/chat/completions`
- Auth header: `Authorization: Bearer ${apiKey}` (was `X-API-Key: ${apiKey}`)
- Remove `X-Agent-Name` and `X-Agent-Version` headers (Bankr-specific)
- Error message: "Venice API error" (was "Bankr API error")

```typescript
async function callLLM(params: {
  apiKey: string;
  model: string;
  messages: LLMMessage[];
  temperature?: number;
  maxTokens?: number;
}): Promise<LLMResponse> {
  const { apiKey, model, messages, temperature = 0.2, maxTokens = 1024 } = params;

  const body: LLMRequest = {
    model,
    messages,
    temperature,
    max_tokens: maxTokens,
    response_format: { type: "json_object" },
  };

  const controller = new AbortController();
  const timeout = setTimeout(() => controller.abort(), REQUEST_TIMEOUT_MS);

  try {
    const res = await fetch(`${VENICE_BASE_URL}/chat/completions`, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Authorization": `Bearer ${apiKey}`,
      },
      body: JSON.stringify(body),
      signal: controller.signal,
    });

    if (!res.ok) {
      const errorText = await res.text().catch(() => "unknown error");
      throw new StrategistError(`Venice API error ${res.status}: ${errorText}`, {
        model,
        status: res.status,
      });
    }

    return (await res.json()) as LLMResponse;
  } finally {
    clearTimeout(timeout);
  }
}
```

- [ ] **Step 4: Update callLLMWithRetry**

Same as `callBankrWithRetry` but calling `callLLM` and logging "Venice" instead of "Bankr".

- [ ] **Step 5: Update deliberate function**

Change `apiKey` usage — it now reads Venice key. Update error messages from "Bankr" to "Venice". The function signature stays the same.

- [ ] **Step 6: Update standalone runner / smoke test at bottom of file**

Change any references to `BANKR_API_KEY` env var to `VENICE_API_KEY`.

- [ ] **Step 7: Update all console.log/warn/error messages**

Replace all `[Strategist] Bankr` with `[Strategist] Venice` throughout the file.

- [ ] **Step 8: Verify build**

```bash
npm run build
```

- [ ] **Step 9: Commit**

```bash
git add src/strategist/index.ts
git commit -m "feat: replace Bankr LLM Gateway with Venice AI — free inference, same OpenAI-compatible format"
```

---

### Task 2: Update Types and Loop for Venice

**Files:**
- Modify: `src/types/index.ts`
- Modify: `src/loop/index.ts`

- [ ] **Step 1: Rename config fields in MurmurConfig**

In `src/types/index.ts`, rename in MurmurConfig:
```typescript
// Old:
  bankrApiKey: string;
  bankrBaseUrl: string;
// New:
  veniceApiKey: string;
  veniceBaseUrl: string;
```

- [ ] **Step 2: Update loop loadConfig**

In `src/loop/index.ts`, update `loadConfig()`:
- `bankrApiKey` → `veniceApiKey`, reading from `process.env.VENICE_API_KEY`
- Remove `bankrBaseUrl` reference
- Update startup warning: "VENICE_API_KEY not set" instead of "BANKR_API_KEY not set"

- [ ] **Step 3: Update loop deliberation call**

In `runDeliberate` function, change `config.bankrApiKey` to `config.veniceApiKey`.

- [ ] **Step 4: Update startup log**

Change the startup message from mentioning Bankr to Venice.

- [ ] **Step 5: Verify build**

```bash
npm run build
```

- [ ] **Step 6: Commit**

```bash
git add src/types/index.ts src/loop/index.ts
git commit -m "feat: update config and loop to use Venice API key instead of Bankr"
```

---

### Task 3: Update .env File

**Files:**
- Modify: `.env`

- [ ] **Step 1: Replace Bankr env vars with Venice**

Remove:
```
BANKR_API_KEY=bk_...
BANKR_BASE_URL=https://llm.bankr.bot
```

Add:
```
VENICE_API_KEY=<user-must-fill>
```

Note: The user needs to get a Venice API key from venice.ai. Leave a placeholder.

- [ ] **Step 2: Enable Filecoin uploads**

Change:
```
SKIP_FILECOIN=true
```
To:
```
SKIP_FILECOIN=false
```

- [ ] **Step 3: No commit** (.env is gitignored)

---

### Task 4: Re-enable Fast-Lane Exits

**Files:**
- Modify: `src/loop/index.ts`

- [ ] **Step 1: Restore the fast-lane check function**

The fast-lane was disabled for testnet testing. Restore it:

```typescript
function checkFastLane(scored: ScoredAsset[]): {
  triggered: boolean;
  asset: ScoredAsset | null;
  reason: string;
} {
  for (const asset of scored) {
    const result = shouldFastLaneExit(asset);
    if (result.triggered) {
      log("risk_gate", `⚡ FAST-LANE: ${result.reason}`);
      return { triggered: true, asset, reason: result.reason };
    }
  }
  return { triggered: false, asset: null, reason: "" };
}
```

- [ ] **Step 2: Verify build**

```bash
npm run build
```

- [ ] **Step 3: Commit**

```bash
git add src/loop/index.ts
git commit -m "fix: re-enable fast-lane exit checks"
```

---

### Task 5: Update README — Remove False Claims, Update Integrations

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Remove bond.credit from prize tracks table**

Delete this row:
```
| **Agents that pay** | bond.credit | Real on-chain trades generate a verifiable credit score via ERC-8004 on Arbitrum |
```

- [ ] **Step 2: Replace Bankr track with Venice AI track**

Change:
```
| **Best Bankr LLM Gateway Use** | Bankr | Bankr routes all LLM calls with dynamic model escalation and on-chain execution |
```
To:
```
| **Private Agents, Trusted Actions** | Venice | Venice AI provides private, no-data-retention LLM inference for all deliberation — the agent's reasoning stays confidential |
```

- [ ] **Step 3: Update architecture section**

Replace all references to "Bankr LLM Gateway" with "Venice AI". Update the 5-Role Committee table:
- Strategist description: change "Uses Bankr LLM Gateway" to "Uses Venice AI"

- [ ] **Step 4: Update the Model Escalation section**

Change:
```
This dynamic routing is powered by **Bankr LLM Gateway** — the Bankr API key manages both model access and on-chain wallet execution.
```
To:
```
LLM inference is powered by **Venice AI** — private, no-data-retention reasoning for all agent deliberation.
```

- [ ] **Step 5: Update Setup section**

Replace:
```
BANKR_API_KEY=your_bankr_key
```
With:
```
VENICE_API_KEY=your_venice_key
```

- [ ] **Step 6: Update the build credit at bottom**

Change "Built with Claude Sonnet 4.6" to reflect Venice AI model if needed.

- [ ] **Step 7: Verify no remaining references to Bankr or bond.credit**

```bash
grep -i "bankr\|bond\.credit" README.md
```

Expected: no matches.

- [ ] **Step 8: Commit**

```bash
git add README.md
git commit -m "docs: update README — replace Bankr with Venice AI, remove bond.credit claim"
```

---

### Task 6: Restore Cooldown to 15 Minutes

**Files:**
- Modify: `src/risk/index.ts`

- [ ] **Step 1: Restore production cooldown**

Change:
```typescript
const DEFAULT_COOLDOWN_MS = 3 * 60 * 1000;    // 3 minutes between trades (testnet)
```
Back to:
```typescript
const DEFAULT_COOLDOWN_MS = 15 * 60 * 1000;   // 15 minutes between trades
```

- [ ] **Step 2: Verify build**

```bash
npm run build
```

- [ ] **Step 3: Commit**

```bash
git add src/risk/index.ts
git commit -m "fix: restore 15-minute trade cooldown"
```

---

### Task 7: End-to-End Verification

- [ ] **Step 1: User provides Venice API key**

The user must sign up at venice.ai and get an API key, then set it in `.env`:
```
VENICE_API_KEY=<key>
```

- [ ] **Step 2: Start the loop**

```bash
SKIP_ON_CHAIN=true npm start
```

Verify:
- Startup shows "Venice" not "Bankr"
- Santiment signals fetch successfully
- Venice AI deliberation returns a decision (not a 403)
- Fast-lane exits work when conditions are met
- Filecoin upload attempts (may fail without FILECOIN_API_TOKEN, that's fine)

- [ ] **Step 3: Start the dashboard**

```bash
npm run dashboard
```

Verify dashboard shows live data with decisions.

- [ ] **Step 4: Commit any fixes**

```bash
git add -A
git commit -m "fix: end-to-end verification fixes"
```
