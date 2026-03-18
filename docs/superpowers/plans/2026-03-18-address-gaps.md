# Address All Gaps Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Close all identified gaps: remove stETH, add on-chain portfolio refresh with USD pricing, fix concentration check, write ERC-8004 registry contract, update position tracking in the loop, and initialize git.

**Architecture:** Remove dead stETH fields from types/loop. Add `refreshPortfolio()` and `getTokenPriceUsd()` to executor using the existing Uniswap Quoter V2. Pass real USD position values into the risk gate concentration check. Write a minimal Solidity ERC-8004 registry contract with Foundry. Wire position add/remove into the loop after execution. Init git and commit after each task.

**Tech Stack:** TypeScript, viem, Uniswap V3 Quoter V2, Solidity, Foundry

---

## File Structure

### Modified files:
- `src/types/index.ts` — Remove `stEthBalance`, `accruedYieldUsd`, Lido config fields
- `src/executor/index.ts` — Add `getTokenPriceUsd()`, `getAllBalances()`, `refreshPortfolio()`
- `src/risk/index.ts` — Fix `checkConcentration()` to accept position USD values; update smoke test
- `src/loop/index.ts` — Use `refreshPortfolio()`, update positions after trades

### New files:
- `contracts/src/AgentRegistry.sol` — ERC-8004 attestation registry
- `contracts/script/DeployAgentRegistry.s.sol` — Foundry deployment script
- `contracts/foundry.toml` — Foundry config
- `contracts/test/AgentRegistry.t.sol` — Solidity tests

---

### Task 1: Initialize Git Repository

**Files:**
- Create: `.git/` (via `git init`)

- [ ] **Step 1: Init git repo**

```bash
cd /home/timidan/Desktop/synth-x
git init
```

- [ ] **Step 2: Create initial commit with all existing code**

```bash
git add -A
git commit -m "feat: initial Murmur codebase — autonomous DeFi operator on Base"
```

---

### Task 2: Remove stETH and Lido References

**Files:**
- Modify: `src/types/index.ts:214-221` (TreasuryState), `src/types/index.ts:269-273` (MurmurConfig)
- Modify: `src/loop/index.ts:112,133,146` (refreshTreasuryState mock, real, catch fallback)
- Modify: `src/risk/index.ts:693-699` (smoke test mock treasury)
- Modify: `.env:22-23` (remove Lido env stubs)

- [ ] **Step 1: Remove stEthBalance and accruedYieldUsd from TreasuryState**

In `src/types/index.ts`, change TreasuryState to:

```typescript
export interface TreasuryState {
  usdcBalance: bigint;
  totalPortfolioUsd: number;
  positions: Position[];
  lastUpdatedAt: string;
}
```

- [ ] **Step 2: Remove Lido fields from MurmurConfig**

In `src/types/index.ts`, remove these lines from MurmurConfig:

```typescript
  // Lido stETH Treasury
  lidoTreasuryAddress: `0x${string}`;
  stEthAddress: `0x${string}`;
  wstEthAddress: `0x${string}`;
```

- [ ] **Step 3: Update loop refreshTreasuryState — remove stEthBalance/accruedYieldUsd**

In `src/loop/index.ts`, update all three locations: the mock return (line ~112), the real return (line ~133), and the catch fallback (line ~146) to remove `stEthBalance` and `accruedYieldUsd`.

- [ ] **Step 4: Update risk smoke test mock treasury**

In `src/risk/index.ts`, remove `stEthBalance` and `accruedYieldUsd` from the mockTreasury object.

- [ ] **Step 5: Remove Lido env stubs from .env**

In `.env`, remove lines 22-23 (`# Lido / stETH` and `LIDO_RPC_URL=`).

- [ ] **Step 6: Verify build passes**

```bash
npm run build
```

- [ ] **Step 7: Commit**

```bash
git add src/types/index.ts src/loop/index.ts src/risk/index.ts .env
git commit -m "refactor: remove stETH/Lido references from TreasuryState and config"
```

---

### Task 3: Add Portfolio Refresh to Executor

**Files:**
- Modify: `src/executor/index.ts` — Add `getTokenPriceUsd()`, `getAllBalances()`, `refreshPortfolio()`

- [ ] **Step 1: Add getTokenPriceUsd function**

Add after `getUsdcBalance` in `src/executor/index.ts`. Uses Quoter V2 to quote 1 unit of token → USDC:

```typescript
export async function getTokenPriceUsd(
  publicClient: PublicClient,
  slug: AssetSlug,
): Promise<number> {
  const token = BASE_TOKENS[slug];
  if (!token) return 0;

  const oneUnit = 10n ** BigInt(token.decimals);
  const fee = resolvePoolFee(slug);

  try {
    const result = await publicClient.simulateContract({
      address: UNISWAP_QUOTER_V2,
      abi: QUOTER_V2_ABI,
      functionName: "quoteExactInputSingle",
      args: [
        {
          tokenIn: token.address,
          tokenOut: BASE_TOKENS.usdc.address,
          amountIn: oneUnit,
          fee,
          sqrtPriceLimitX96: BigInt(0),
        },
      ],
    });

    const [amountOut] = result.result as [bigint, bigint, number, bigint];
    return Number(amountOut) / 1e6; // USDC has 6 decimals
  } catch {
    return 0;
  }
}
```

- [ ] **Step 2: Add getAllBalances function**

Returns a map of slug → { balance, usdValue } for all assets the agent holds:

Note: "ethereum" and "weth" in `BASE_TOKENS` share the same address (`0x4200...0006`). We must deduplicate by address to avoid double-counting the WETH balance.

```typescript
export async function getAllBalances(
  publicClient: PublicClient,
  ownerAddress: Address,
): Promise<Map<AssetSlug, { balance: bigint; usdValue: number }>> {
  const results = new Map<AssetSlug, { balance: bigint; usdValue: number }>();

  const slugs = Object.keys(BASE_TOKENS).filter(
    (k) => k !== "usdc",
  ) as AssetSlug[];

  // Deduplicate slugs that share the same token address (e.g. "ethereum" and "weth")
  const seenAddresses = new Set<string>();
  const uniqueSlugs = slugs.filter((slug) => {
    const addr = BASE_TOKENS[slug]?.address.toLowerCase();
    if (!addr || seenAddresses.has(addr)) return false;
    seenAddresses.add(addr);
    return true;
  });

  await Promise.all(
    uniqueSlugs.map(async (slug) => {
      const token = BASE_TOKENS[slug];
      if (!token) return;

      try {
        const balance = await getTokenBalance(
          publicClient,
          token.address,
          ownerAddress,
        );

        if (balance > 0n) {
          const priceUsd = await getTokenPriceUsd(publicClient, slug);
          const usdValue =
            (Number(balance) / 10 ** token.decimals) * priceUsd;
          results.set(slug, { balance, usdValue });
        }
      } catch {
        // skip assets that fail to query
      }
    }),
  );

  return results;
}
```

- [ ] **Step 3: Add refreshPortfolio function**

Builds a complete TreasuryState from on-chain data. First, merge `TreasuryState` and `Position` into the existing import from `../types/index.js` (which already imports `AssetSlug`, `ExecutionResult`, etc.):

```typescript
// Merge into existing import at top of file:
import {
  type AssetSlug,
  type ExecutionResult,
  type LLMDecision,
  type RiskGateResult,
  type UniswapQuote,
  type TreasuryState,
  type Position,
  MurmurError,
} from "../types/index.js";
```

Then add the function after `getAllBalances`:

```typescript
export async function refreshPortfolio(
  publicClient: PublicClient,
  ownerAddress: Address,
): Promise<TreasuryState> {
  const usdcBalance = await getUsdcBalance(publicClient, ownerAddress);
  const usdcUsd = Number(usdcBalance) / 1e6;

  const balances = await getAllBalances(publicClient, ownerAddress);

  const positions: Position[] = [];
  let positionsUsd = 0;

  for (const [slug, { balance, usdValue }] of balances) {
    const token = BASE_TOKENS[slug];
    if (!token) continue;

    positions.push({
      slug,
      tokenAddress: token.address,
      amountHeld: balance,
      usdValueAtEntry: usdValue, // current value, not entry — best we have without tracking
      entryTxHash: "0x0000000000000000000000000000000000000000000000000000000000000000",
      entryAt: new Date().toISOString(),
      thesis: "reconstructed from on-chain balance",
      invalidationCondition: "",
    });

    positionsUsd += usdValue;
  }

  return {
    usdcBalance,
    totalPortfolioUsd: usdcUsd + positionsUsd,
    positions,
    lastUpdatedAt: new Date().toISOString(),
  };
}
```

- [ ] **Step 4: Verify build passes**

```bash
npm run build
```

- [ ] **Step 5: Commit**

```bash
git add src/executor/index.ts
git commit -m "feat: add on-chain portfolio refresh with USD pricing via Uniswap quoter"
```

---

### Task 4: Fix Concentration Check in Risk Gate

**Files:**
- Modify: `src/risk/index.ts:260-301` (checkConcentration function)

- [ ] **Step 1: Update checkConcentration to use position USD values**

Replace the simplified `Number(existingPosition.amountHeld) * 1e-18 * 1` with `existingPosition.usdValueAtEntry` since positions now carry USD values from the portfolio refresh:

```typescript
function checkConcentration(
  slug: AssetSlug,
  effectiveSizeUsd: number,
  action: TradeAction,
  treasury: TreasuryState,
  policy: RiskPolicy,
): RiskCheck {
  if (action !== "buy") {
    return check(
      "concentration",
      true,
      `Action "${action}" — concentration check not applicable`,
    );
  }

  const totalPortfolioUsd = treasury.totalPortfolioUsd;
  if (totalPortfolioUsd <= 0) {
    return check("concentration", true, "Portfolio value is zero — no concentration risk");
  }

  const existingPosition = treasury.positions.find((p) => p.slug === slug);
  const existingUsd = existingPosition ? existingPosition.usdValueAtEntry : 0;

  const projectedExposurePct =
    ((existingUsd + effectiveSizeUsd) / totalPortfolioUsd) * 100;

  const fits = projectedExposurePct <= policy.maxConcentrationPct;
  return check(
    "concentration",
    fits,
    fits
      ? `Projected ${slug} exposure ${projectedExposurePct.toFixed(1)}% within ${policy.maxConcentrationPct}% limit`
      : `Projected ${slug} exposure ${projectedExposurePct.toFixed(1)}% exceeds ${policy.maxConcentrationPct}% limit`,
    fits
      ? projectedExposurePct > policy.maxConcentrationPct * 0.8
        ? "warn"
        : "pass"
      : "fail",
  );
}
```

- [ ] **Step 2: Verify build passes**

```bash
npm run build
```

- [ ] **Step 3: Commit**

```bash
git add src/risk/index.ts
git commit -m "fix: use real USD values in concentration check instead of simplified token amount"
```

---

### Task 5: Wire Portfolio Refresh and Position Tracking Into Loop

**Files:**
- Modify: `src/loop/index.ts` — Use `refreshPortfolio`, update positions after trades

- [ ] **Step 1: Import refreshPortfolio from executor**

In `src/loop/index.ts`, add `refreshPortfolio` to the executor import:

```typescript
import {
  execute,
  getUsdcBalance,
  createClients,
  BASE_TOKENS,
  refreshPortfolio,
} from "../executor/index.js";
```

- [ ] **Step 2: Rewrite refreshTreasuryState to use refreshPortfolio**

Replace the existing `refreshTreasuryState` function:

```typescript
async function refreshTreasuryState(
  config: ReturnType<typeof loadConfig>,
): Promise<TreasuryState> {
  if (!config.agentPrivateKey || !config.agentAddress) {
    return {
      usdcBalance: BigInt(1000 * 1e6),
      totalPortfolioUsd: 1000,
      positions: treasuryState?.positions ?? [],
      lastUpdatedAt: new Date().toISOString(),
    };
  }

  try {
    const { publicClient } = createClients({
      rpcUrl: config.baseRpcUrl,
      privateKey: config.agentPrivateKey,
    });

    return await refreshPortfolio(publicClient, config.agentAddress);
  } catch (err) {
    console.warn(
      `[Loop] Could not refresh treasury state: ${(err as Error).message}`,
    );
    return (
      treasuryState ?? {
        usdcBalance: BigInt(0),
        totalPortfolioUsd: 0,
        positions: [],
        lastUpdatedAt: new Date().toISOString(),
      }
    );
  }
}
```

- [ ] **Step 3: Refresh treasury after execution**

In `runCycle`, after the execute phase and before attest, add a treasury refresh:

```typescript
    // ── 9.5 Refresh treasury after trade
    if (executionResult) {
      treasuryState = await refreshTreasuryState(config);
      log("execute", `Treasury refreshed — $${treasuryState.totalPortfolioUsd.toFixed(2)} total | ${treasuryState.positions.length} positions`);
    }
```

- [ ] **Step 4: Verify build passes**

```bash
npm run build
```

- [ ] **Step 5: Commit**

```bash
git add src/loop/index.ts
git commit -m "feat: wire on-chain portfolio refresh into loop with post-trade position update"
```

---

### Task 6: ERC-8004 Agent Registry Solidity Contract

**Files:**
- Create: `contracts/foundry.toml`
- Create: `contracts/src/AgentRegistry.sol`
- Create: `contracts/test/AgentRegistry.t.sol`
- Create: `contracts/script/DeployAgentRegistry.s.sol`

- [ ] **Step 1: Create Foundry config**

```toml
[profile.default]
src = "src"
out = "out"
libs = ["lib"]
solc_version = "0.8.24"
optimizer = true
optimizer_runs = 200
evm_version = "cancun"

[profile.default.fmt]
line_length = 100
```

- [ ] **Step 2: Write AgentRegistry.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title AgentRegistry — ERC-8004 Attestation Registry for Autonomous Agents
/// @notice Stores immutable attestation records for agent decision receipts on Base Mainnet.
///         Each attestation links a keccak256 receipt hash to an IPFS/Filecoin metadata URI.
contract AgentRegistry {
    struct Attestation {
        bytes32 receiptHash;
        string metadataUri;
        string actionType;
        uint256 timestamp;
        uint256 attestationId;
    }

    uint256 private _nextId = 1;

    /// @dev agent address => list of attestations
    mapping(address => Attestation[]) private _attestations;

    /// @dev receiptHash => attesting agent (prevents duplicate attestations for same hash)
    mapping(bytes32 => address) public attestedBy;

    event AgentAttested(
        address indexed agent,
        bytes32 indexed receiptHash,
        uint256 attestationId,
        string metadataUri,
        string actionType
    );

    error AlreadyAttested(bytes32 receiptHash, address existingAgent);
    error EmptyReceiptHash();
    error EmptyMetadataUri();

    /// @notice Record an attestation for a decision receipt.
    /// @param receiptHash Keccak256 hash of the canonical receipt JSON.
    /// @param metadataUri IPFS/Filecoin URI pointing to the full receipt (e.g. "ipfs://Qm...").
    /// @param actionType The action taken — "buy", "exit", "reduce", or "hold".
    /// @return attestationId The unique ID of this attestation.
    function attest(
        bytes32 receiptHash,
        string calldata metadataUri,
        string calldata actionType
    ) external returns (uint256 attestationId) {
        if (receiptHash == bytes32(0)) revert EmptyReceiptHash();
        if (bytes(metadataUri).length == 0) revert EmptyMetadataUri();
        if (attestedBy[receiptHash] != address(0)) {
            revert AlreadyAttested(receiptHash, attestedBy[receiptHash]);
        }

        attestationId = _nextId++;
        attestedBy[receiptHash] = msg.sender;

        _attestations[msg.sender].push(
            Attestation({
                receiptHash: receiptHash,
                metadataUri: metadataUri,
                actionType: actionType,
                timestamp: block.timestamp,
                attestationId: attestationId
            })
        );

        emit AgentAttested(msg.sender, receiptHash, attestationId, metadataUri, actionType);
    }

    /// @notice Get all attestations for a given agent.
    function getAttestations(address agent) external view returns (Attestation[] memory) {
        return _attestations[agent];
    }

    /// @notice Get the number of attestations for a given agent.
    function attestationCount(address agent) external view returns (uint256) {
        return _attestations[agent].length;
    }
}
```

- [ ] **Step 3: Write Foundry tests**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import "../src/AgentRegistry.sol";

contract AgentRegistryTest is Test {
    AgentRegistry registry;
    address agent = address(0xBEEF);

    function setUp() public {
        registry = new AgentRegistry();
    }

    function test_attest_emits_event() public {
        bytes32 hash = keccak256("receipt-1");
        vm.prank(agent);
        vm.expectEmit(true, true, false, true);
        emit AgentRegistry.AgentAttested(agent, hash, 1, "ipfs://Qm1", "buy");
        registry.attest(hash, "ipfs://Qm1", "buy");
    }

    function test_attest_stores_attestation() public {
        bytes32 hash = keccak256("receipt-1");
        vm.prank(agent);
        uint256 id = registry.attest(hash, "ipfs://Qm1", "buy");

        assertEq(id, 1);
        assertEq(registry.attestedBy(hash), agent);

        AgentRegistry.Attestation[] memory atts = registry.getAttestations(agent);
        assertEq(atts.length, 1);
        assertEq(atts[0].receiptHash, hash);
        assertEq(atts[0].metadataUri, "ipfs://Qm1");
        assertEq(atts[0].actionType, "buy");
        assertEq(atts[0].attestationId, 1);
    }

    function test_attest_reverts_on_duplicate_hash() public {
        bytes32 hash = keccak256("receipt-dup");
        vm.prank(agent);
        registry.attest(hash, "ipfs://Qm1", "buy");

        vm.prank(agent);
        vm.expectRevert(
            abi.encodeWithSelector(AgentRegistry.AlreadyAttested.selector, hash, agent)
        );
        registry.attest(hash, "ipfs://Qm2", "hold");
    }

    function test_attest_reverts_on_empty_hash() public {
        vm.prank(agent);
        vm.expectRevert(AgentRegistry.EmptyReceiptHash.selector);
        registry.attest(bytes32(0), "ipfs://Qm1", "buy");
    }

    function test_attest_reverts_on_empty_uri() public {
        vm.prank(agent);
        vm.expectRevert(AgentRegistry.EmptyMetadataUri.selector);
        registry.attest(keccak256("receipt"), "", "buy");
    }

    function test_multiple_attestations_increment_id() public {
        vm.startPrank(agent);
        uint256 id1 = registry.attest(keccak256("r1"), "ipfs://1", "buy");
        uint256 id2 = registry.attest(keccak256("r2"), "ipfs://2", "hold");
        uint256 id3 = registry.attest(keccak256("r3"), "ipfs://3", "exit");
        vm.stopPrank();

        assertEq(id1, 1);
        assertEq(id2, 2);
        assertEq(id3, 3);
        assertEq(registry.attestationCount(agent), 3);
    }

    function test_different_agents_can_attest_different_hashes() public {
        address agent2 = address(0xCAFE);
        bytes32 hash1 = keccak256("r1");
        bytes32 hash2 = keccak256("r2");

        vm.prank(agent);
        registry.attest(hash1, "ipfs://1", "buy");

        vm.prank(agent2);
        registry.attest(hash2, "ipfs://2", "exit");

        assertEq(registry.attestationCount(agent), 1);
        assertEq(registry.attestationCount(agent2), 1);
    }
}
```

- [ ] **Step 4: Write deployment script**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Script.sol";
import "../src/AgentRegistry.sol";

contract DeployAgentRegistry is Script {
    function run() external {
        uint256 deployerPrivateKey = vm.envUint("DEPLOYER_PRIVATE_KEY");
        vm.startBroadcast(deployerPrivateKey);

        AgentRegistry registry = new AgentRegistry();
        console.log("AgentRegistry deployed at:", address(registry));

        vm.stopBroadcast();
    }
}
```

- [ ] **Step 5: Install Foundry deps and run tests**

```bash
cd /home/timidan/Desktop/synth-x/contracts
forge install foundry-rs/forge-std --no-commit
forge test -vv
```

Expected: All 6 tests pass.

- [ ] **Step 6: Commit**

```bash
cd /home/timidan/Desktop/synth-x
git add contracts/
git commit -m "feat: add ERC-8004 AgentRegistry Solidity contract with Foundry tests and deploy script"
```

---

### Task 7: Update Notary Placeholder Address Comment

**Files:**
- Modify: `src/notary/index.ts:48-49`

- [ ] **Step 1: Update the placeholder comment**

Change the comment to reference the deployment script:

```typescript
const ERC8004_REGISTRY_ADDRESS: Address =
  "0x0000000071727De22E5E9d8BAf0edAc6f37da032"; // deploy with: cd contracts && forge script script/DeployAgentRegistry.s.sol --rpc-url $BASE_RPC_URL --broadcast
```

- [ ] **Step 2: Verify build passes**

```bash
npm run build
```

- [ ] **Step 3: Commit**

```bash
git add src/notary/index.ts
git commit -m "docs: update ERC-8004 registry address comment with deployment instructions"
```
