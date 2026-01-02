# AI Lab Simulator (Browser, Single Player) - Architecture Plan

This document describes a React + Web Worker architecture optimized for:
- Smooth UI (DOM tycoon dashboard)
- Deterministic, testable simulation
- Clear boundaries that are easy for humans and AI tools to modify safely

## Product Constraints (Locked In)
- Single player, offline only
- Real-time with `pause` + speed controls (`1x`, `2x`, `5x`)
- Simulated training (no WebGPU/WASM training)
- Hundreds of entities, dozens of competitors
- Game mechanisms operate on **Game Days** as the smallest meaningful unit
- When the tab is hidden/unfocused: automatically pause; do not auto-resume
- Save slots + autosaves
- Debug mode supports exporting a replay JSON

## Big Architectural Decisions

### 1) Functional Core / Imperative Shell
- **Simulation core** is pure TypeScript (no DOM, no timers, no randomness via `Math.random`).
- **Runtime shell** (Worker) supplies time progression and message IO.
- **UI** (React) sends player intent as commands and renders read-only views.

### 2) Simulation Is Authoritative In a Web Worker
- Worker runs the tick loop and owns the canonical `SimState`.
- UI thread never mutates `SimState` directly.
- Worker emits periodic `ViewSnapshot` updates (coarser than sim ticks).

### 3) Determinism by Contract
Determinism is required for debugging, AI-assisted development, and replay.

Rules:
- Only inputs to the sim are:
  - initial seed
  - initial state (or "new game" config)
  - the ordered command log (commands applied at specific ticks)
- Use a seeded PRNG module (`rng.next()`), never `Math.random()`.
- Avoid reading wall clock (`Date.now`) inside the sim core.
- Avoid data structures with non-obvious iteration order in hot paths.
  - Prefer arrays + stable sorting when order matters.

## Time Model

### Concepts
- **Real time**: actual milliseconds passing in the browser.
- **Sim ticks**: fixed-step simulation steps.
- **Game days**: the smallest unit used by game mechanics.

### Suggested configuration (tunable)
- `TICK_REAL_MS`: how often the Worker attempts to step (e.g. 250ms-2000ms)
- `TICKS_PER_DAY`: integer (e.g. 3-10)
- `DAYS_PER_MONTH = 30`
- `MONTHS_PER_YEAR = 12`

A "Game Day" occurs when `tick % TICKS_PER_DAY === 0`.

### Why ticks at all if days are smallest unit?
- Ticks give smooth progress bars and predictable pacing at 1x/2x/5x.
- Mechanics that truly only change daily should still update on day boundaries.

### Pause on tab hidden
- UI detects `document.visibilityState !== 'visible'` and sends `SetPaused(true)`.
- When visible again, remain paused until player clicks resume.

## Simulation Core Design

### Modules
- `state/`: data types, initialization, invariants
- `commands/`: the only way to mutate state
- `systems/`: pure functions that advance the simulation
- `engine/`: tick orchestration, scheduling, RNG

### State Shape (normalized)
Keep state normalized for performance and diffability.

Example skeleton:

```ts
export type SimState = {
  version: number;
  tick: number;
  day: number;

  rng: RngState;

  player: {
    money: number;
    reputation: number;
    compute: {
      owned: number;
      allocated: number;
    };
  };

  staffById: Record<string, Staff>;
  jobsById: Record<string, Job>;
  modelsById: Record<string, Model>;
  productsById: Record<string, Product>;

  market: MarketState;
  competitorsById: Record<string, Competitor>;

  // Scheduling for coarse updates (competitors, market cycles)
  scheduledEvents: ScheduledEventQueue;
};
```

Guidelines:
- Prefer `Record<Id, Entity>` + separate `allIds: Id[]` when stable ordering is needed.
- Store only "source of truth" in the sim. Derived UI values belong in `ViewSnapshot`.

### ViewSnapshot (UI Data Contract)

The `ViewSnapshot` is the read-only projection of `SimState` sent to the UI. It contains pre-computed derived values to avoid redundant calculations in React.

```ts
export type ViewSnapshot = {
  // Metadata
  snapshotVersion: number;
  generatedAtTick: number;

  // Time
  tick: number;
  day: number;
  month: number;
  year: number;
  isPaused: boolean;
  speed: 0 | 1 | 2 | 5;

  // Player summary (derived)
  player: {
    money: number;
    moneyDelta: number; // change since last day (for trend indicators)
    reputation: number;
    computeOwned: number;
    computeAllocated: number;
    computeAvailable: number; // derived: owned - allocated
  };

  // Entity lists (UI-ready, sorted)
  staff: StaffView[];
  jobs: JobView[];
  models: ModelView[];
  products: ProductView[];

  // Market overview
  market: {
    segments: MarketSegmentView[];
    totalAddressableMarket: number;
    playerMarketShare: number;
  };

  // Competition
  competitors: CompetitorView[];
  leaderboard: LeaderboardEntry[];

  // Notifications / Events
  recentEvents: GameEventView[];
  alerts: AlertView[];

  // Charts (updated less frequently, e.g., daily)
  charts?: {
    revenueHistory: DataPoint[];
    reputationHistory: DataPoint[];
    marketShareHistory: DataPoint[];
  };
};

// Example entity views (slimmed for UI)
export type StaffView = {
  id: string;
  name: string;
  role: string;
  salary: number;
  productivity: number;
  assignedJobId: string | null;
};

export type JobView = {
  id: string;
  type: 'training' | 'research' | 'deployment';
  name: string;
  progress: number; // 0-1
  estimatedDaysRemaining: number;
  assignedStaffCount: number;
  computeAllocated: number;
};

export type CompetitorView = {
  id: string;
  name: string;
  reputation: number;
  marketShare: number;
  threat: 'low' | 'medium' | 'high';
};

export type DataPoint = { day: number; value: number };
```

Guidelines:
- Keep `ViewSnapshot` flat where possible; avoid deep nesting.
- Pre-sort lists (e.g., staff by name, jobs by progress) in the Worker.
- Include `*Delta` fields for values where the UI shows trends.
- `charts` is optional and updated only on day boundaries to reduce overhead.

### Commands
Commands encode player intent and are applied deterministically.

```ts
export type GameCommand =
  | { type: 'SetSpeed'; speed: 0 | 1 | 2 | 5 }
  | { type: 'SetPaused'; paused: boolean }
  | { type: 'HireStaff'; roleId: string }
  | { type: 'StartTrainingJob'; projectId: string; computeUnits: number }
  | { type: 'LaunchProduct'; modelId: string; marketId: string }
  | { type: 'SetPricing'; productId: string; price: number };
```

Rules:
- Commands must be validated (and rejected with a reason) before applying.
- A command applied at tick `T` must yield the same result during replay.

### Systems
Implement sim progression as a list of small, focused systems.

Example system categories:
- Finance: payroll, expenses, revenue
- Training: job progress, compute usage, model quality updates
- Market: demand curves, pricing response, churn
- Competitors: scheduled decisions (daily/weekly)

Suggested structure:
- `systems/*.ts` exports `run(state, ctx): { state, events }`
- `ctx` contains deterministic inputs for that tick/day (e.g. rng instance, config)

### Scheduling: "not every tick"
For competitors and market, avoid heavy work every tick.
- Run competitor decision-making at day boundaries (or every N days).
- Use `scheduledEvents` to run specific actions on a future day.

## Competitor AI System

Competitors provide dynamic challenge and prevent optimal static strategies. The AI should feel intentional but remain performant and deterministic.

### Competitor State

```ts
export type Competitor = {
  id: string;
  name: string;
  
  // Core stats
  money: number;
  reputation: number;
  computeCapacity: number;
  
  // Personality (immutable, set at game start)
  personality: CompetitorPersonality;
  
  // Current focus
  strategy: CompetitorStrategy;
  strategyLockedUntilDay: number; // prevents thrashing
  
  // Assets
  modelIds: string[];
  productIds: string[];
  staffCount: number;
  
  // Decision state (for debugging/replay)
  lastDecisionDay: number;
  lastDecisionReason: string;
};

export type CompetitorPersonality = {
  aggressiveness: number;    // 0-1: pricing, expansion speed
  innovationFocus: number;   // 0-1: new models vs. optimizing existing
  riskTolerance: number;     // 0-1: willingness to overspend
  reactionSpeed: number;     // 0-1: how quickly they respond to player
};

export type CompetitorStrategy =
  | { type: 'expand'; targetSegmentId: string }
  | { type: 'defend'; segmentId: string }
  | { type: 'undercut'; targetCompetitorId: string }
  | { type: 'innovate'; researchFocus: string }
  | { type: 'consolidate' }; // cut costs, stabilize
```

### Decision Frequency

Competitors don't decide every tick—that's expensive and unrealistic.

```ts
export const COMPETITOR_CONFIG = {
  // Major strategic decisions
  strategyReviewIntervalDays: 7,
  strategyMinLockDays: 14, // prevent thrashing
  
  // Tactical adjustments (pricing, allocation)
  tacticalReviewIntervalDays: 1,
  
  // Market scanning (awareness of player actions)
  awarenessUpdateIntervalDays: 3,
};
```

### Decision Pipeline

Run on day boundaries:

```ts
export function runCompetitorSystem(
  state: SimState,
  ctx: { rng: Rng; config: GameConfig; currentDay: number }
): { state: SimState; events: GameEvent[] } {
  const events: GameEvent[] = [];
  let newState = state;

  for (const competitorId of Object.keys(state.competitorsById)) {
    const competitor = state.competitorsById[competitorId];
    
    // Skip if not decision day
    if (ctx.currentDay % COMPETITOR_CONFIG.tacticalReviewIntervalDays !== 0) {
      continue;
    }

    // 1. Update awareness (what do they know about market/player?)
    const awareness = computeAwareness(competitor, state, ctx);

    // 2. Evaluate current strategy
    const strategyScore = evaluateCurrentStrategy(competitor, awareness, ctx);

    // 3. Maybe switch strategy (if unlocked and score is poor)
    if (
      ctx.currentDay >= competitor.strategyLockedUntilDay &&
      strategyScore < 0.4 &&
      ctx.currentDay % COMPETITOR_CONFIG.strategyReviewIntervalDays === 0
    ) {
      const { newStrategy, reason } = selectNewStrategy(competitor, awareness, ctx);
      newState = updateCompetitor(newState, competitorId, {
        strategy: newStrategy,
        strategyLockedUntilDay: ctx.currentDay + COMPETITOR_CONFIG.strategyMinLockDays,
        lastDecisionDay: ctx.currentDay,
        lastDecisionReason: reason,
      });
      events.push({ type: 'competitorStrategyChange', competitorId, strategy: newStrategy });
    }

    // 4. Execute tactical actions based on strategy
    const actions = executeTactics(competitor, awareness, ctx);
    newState = applyCompetitorActions(newState, competitorId, actions);
  }

  return { state: newState, events };
}
```

### Strategy Selection Logic

Strategies are chosen based on market position and personality:

```ts
function selectNewStrategy(
  competitor: Competitor,
  awareness: CompetitorAwareness,
  ctx: { rng: Rng }
): { newStrategy: CompetitorStrategy; reason: string } {
  const dominated = awareness.segmentsWhereLeading.length === 0;
  const threatened = awareness.playerGrowthRate > 0.1;
  const cashRich = competitor.money > awareness.averageCompetitorMoney * 1.5;
  const cashPoor = competitor.money < awareness.averageCompetitorMoney * 0.5;

  // Build weighted options based on situation + personality
  const options: Array<{ strategy: CompetitorStrategy; weight: number; reason: string }> = [];

  if (dominated && cashRich) {
    options.push({
      strategy: { type: 'expand', targetSegmentId: awareness.bestOpportunitySegment },
      weight: 0.5 + competitor.personality.aggressiveness * 0.3,
      reason: 'No market presence, expanding aggressively',
    });
  }

  if (threatened && awareness.playerWeakestSegment) {
    options.push({
      strategy: { type: 'undercut', targetCompetitorId: 'player' },
      weight: 0.3 + competitor.personality.aggressiveness * 0.4,
      reason: 'Player growing fast, undercutting to slow them',
    });
  }

  if (cashPoor) {
    options.push({
      strategy: { type: 'consolidate' },
      weight: 0.7 - competitor.personality.riskTolerance * 0.3,
      reason: 'Low funds, consolidating',
    });
  }

  if (competitor.personality.innovationFocus > 0.6) {
    options.push({
      strategy: { type: 'innovate', researchFocus: 'next-gen' },
      weight: competitor.personality.innovationFocus * 0.5,
      reason: 'Innovation-focused personality',
    });
  }

  // Default fallback
  if (options.length === 0) {
    options.push({
      strategy: { type: 'defend', segmentId: awareness.strongestSegment },
      weight: 1,
      reason: 'Default: defending strongest position',
    });
  }

  // Weighted random selection (deterministic via seeded RNG)
  return weightedRandomSelect(options, ctx.rng);
}
```

### Tactical Actions

Each strategy maps to concrete actions:

```ts
type TacticalAction =
  | { type: 'adjustPricing'; productId: string; delta: number }
  | { type: 'allocateCompute'; jobId: string; amount: number }
  | { type: 'launchProduct'; modelId: string; segmentId: string }
  | { type: 'cancelProject'; jobId: string }
  | { type: 'hireStaff'; count: number }
  | { type: 'layoffStaff'; count: number };

function executeTactics(
  competitor: Competitor,
  awareness: CompetitorAwareness,
  ctx: { rng: Rng }
): TacticalAction[] {
  switch (competitor.strategy.type) {
    case 'undercut':
      // Lower prices on competing products
      return competitor.productIds
        .filter((pid) => awareness.competingProducts.includes(pid))
        .map((productId) => ({
          type: 'adjustPricing' as const,
          productId,
          delta: -0.05 - ctx.rng.next() * 0.1, // 5-15% reduction
        }));

    case 'expand':
      // Allocate more compute to training, prepare new product
      return [
        { type: 'allocateCompute', jobId: 'expansion-training', amount: competitor.computeCapacity * 0.6 },
        { type: 'hireStaff', count: Math.floor(1 + ctx.rng.next() * 2) },
      ];

    case 'consolidate':
      // Cut costs
      return [
        { type: 'layoffStaff', count: Math.max(1, Math.floor(competitor.staffCount * 0.1)) },
        { type: 'cancelProject', jobId: awareness.lowestPriorityJob },
      ];

    // ... other strategies
    default:
      return [];
  }
}
```

### Performance Considerations

- **Batch processing**: Process all competitors in a single system run, not individually.
- **Early exit**: Skip competitors whose decision day hasn't arrived.
- **Precomputed awareness**: Cache market metrics once per day, share across competitors.
- **Personality diversity**: Vary decision intervals slightly per competitor to spread load.

### Debugging & Tuning

```ts
// Include in ViewSnapshot for debug UI
export type CompetitorDebugView = {
  id: string;
  currentStrategy: CompetitorStrategy;
  strategyScore: number;
  lastDecisionReason: string;
  awarenessSnapshot: CompetitorAwareness;
};
```

Log strategy changes as `GameEvent` for replay analysis. In debug mode, show competitor decision trees in the UI.

## Worker Runtime

### Responsibilities
- Own the authoritative state
- Manage time progression and speed
- Apply commands in a deterministic order
- Emit view updates at a controlled cadence

### Tick loop approach
- Maintain an accumulator based on real time.
- Advance `N` sim ticks per loop depending on speed.
- Clamp maximum ticks processed per loop to avoid UI starvation.

Important: determinism for debugging comes from replaying with recorded tick IDs,
not from matching wall clock.

### ViewSnapshot frequency
Do not postMessage on every tick.
- Sim tick: e.g. 1-4 times per second
- ViewSnapshot: e.g. 5-10 times per second max, often less
- Heavy aggregates (charts): e.g. once per game day

### Protocol (UI <-> Worker)
Use a versioned message protocol with request IDs.

```ts
export type ClientMessage =
  | { v: 1; type: 'command'; id: string; atTick?: number; command: GameCommand }
  | { v: 1; type: 'requestView'; id: string }
  | { v: 1; type: 'save'; id: string; slotId: string }
  | { v: 1; type: 'load'; id: string; slotId: string }
  | { v: 1; type: 'exportReplay'; id: string };

export type WorkerMessage =
  | { v: 1; type: 'ack'; id: string; appliedAtTick: number }
  | { v: 1; type: 'reject'; id: string; reason: string }
  | { v: 1; type: 'view'; view: ViewSnapshot }
  | { v: 1; type: 'saveDone'; slotId: string }
  | { v: 1; type: 'replayExport'; replay: ReplayFile };
```

Notes:
- Worker assigns `appliedAtTick` so replay has a stable timeline.
- In dev, runtime-validate messages (optional) to catch mismatches early.

## UI Architecture (React)

### Principles
- Treat Worker as a "server".
- Keep React state minimal: mostly UI state (panel open, filters) and the latest `ViewSnapshot`.

### State management
- Use an external store that supports `useSyncExternalStore` (or Zustand).
- Store `ViewSnapshot` and derive slices via selectors.

### Rendering performance
- Memoize expensive derived data.
- Virtualize long lists (staff/jobs).
- Batch UI updates to snapshot cadence.

### Charts
Prefer simple, fast chart libs or minimal custom SVG.
- Update charts on day boundaries when possible.

## Persistence

### Save format
- Store `SimState` plus metadata.

```ts
export type SaveFile = {
  saveVersion: number;
  createdAtIso: string;
  updatedAtIso: string;
  slotId: string;
  sim: SimState;
};
```

### Autosaves
- Autosave every 30-60 seconds real time OR on major milestones.
- Keep a rolling set (e.g. last 5 autosaves) per slot.

### Storage
- Use IndexedDB (best for structured data + size).
- Version saves with migrations (`saveVersion`).

### Save Migrations

When `saveVersion` changes, apply sequential migrations to upgrade old saves.

```ts
export type Migration = {
  fromVersion: number;
  toVersion: number;
  migrate: (oldSave: unknown) => unknown;
};

// Registry of all migrations, in order
export const migrations: Migration[] = [
  {
    fromVersion: 1,
    toVersion: 2,
    migrate: (save: any) => ({
      ...save,
      saveVersion: 2,
      sim: {
        ...save.sim,
        // Example: add new field with default
        player: { ...save.sim.player, reputation: save.sim.player.reputation ?? 50 },
      },
    }),
  },
  {
    fromVersion: 2,
    toVersion: 3,
    migrate: (save: any) => ({
      ...save,
      saveVersion: 3,
      sim: {
        ...save.sim,
        // Example: rename field
        competitorsById: save.sim.rivalsById ?? save.sim.competitorsById,
      },
    }),
  },
];

export function migrateSave(save: SaveFile, targetVersion: number): SaveFile {
  let current = save;
  while (current.saveVersion < targetVersion) {
    const migration = migrations.find((m) => m.fromVersion === current.saveVersion);
    if (!migration) {
      throw new Error(`No migration path from version ${current.saveVersion}`);
    }
    current = migration.migrate(current) as SaveFile;
  }
  return current;
}
```

Guidelines:
- Migrations must be pure and deterministic.
- Never delete a migration; old saves may still exist.
- Test migrations with snapshot fixtures (save v1 -> v2 -> v3).
- If a migration is complex, add a validation step after each transform.
- Log migration events for debugging: `console.info(`Migrated save from v${from} to v${to}`)`.

### Storage Quota & Cleanup

IndexedDB has per-origin storage limits. Implement proactive cleanup:

```ts
export const STORAGE_CONFIG = {
  maxSaveSlotsPerUser: 10,
  maxAutosavesPerSlot: 5,
  maxTotalSizeMB: 50,
  warningThresholdPercent: 80,
};

export async function cleanupOldSaves(db: IDBDatabase): Promise<void> {
  // 1. Delete autosaves beyond the limit (keep newest N)
  // 2. If still over quota, prompt user to delete manual saves
  // 3. Never auto-delete manual saves without confirmation
}

export async function getStorageUsage(): Promise<{ usedMB: number; quotaMB: number }> {
  if (navigator.storage?.estimate) {
    const { usage, quota } = await navigator.storage.estimate();
    return { usedMB: (usage ?? 0) / 1e6, quotaMB: (quota ?? 0) / 1e6 };
  }
  return { usedMB: 0, quotaMB: 50 }; // fallback estimate
}
```

UI should display storage usage and allow manual deletion of saves.

## Error Recovery

### Worker Crash Recovery

The Worker can crash due to infinite loops, memory exhaustion, or unhandled exceptions. The UI must detect and recover gracefully.

```ts
// UI-side worker manager
export class WorkerManager {
  private worker: Worker | null = null;
  private lastHeartbeat: number = 0;
  private heartbeatInterval: number | null = null;
  private readonly HEARTBEAT_TIMEOUT_MS = 5000;

  start(): void {
    this.worker = new Worker(new URL('./sim.worker.ts', import.meta.url), { type: 'module' });
    this.worker.onmessage = this.handleMessage.bind(this);
    this.worker.onerror = this.handleError.bind(this);
    this.startHeartbeat();
  }

  private startHeartbeat(): void {
    this.heartbeatInterval = window.setInterval(() => {
      this.worker?.postMessage({ v: 1, type: 'ping', id: crypto.randomUUID() });
      
      const elapsed = Date.now() - this.lastHeartbeat;
      if (this.lastHeartbeat > 0 && elapsed > this.HEARTBEAT_TIMEOUT_MS) {
        this.handleUnresponsive();
      }
    }, 2000);
  }

  private handleMessage(event: MessageEvent): void {
    if (event.data.type === 'pong') {
      this.lastHeartbeat = Date.now();
      return;
    }
    // ... handle other messages
  }

  private handleError(error: ErrorEvent): void {
    console.error('Worker error:', error);
    this.recoverFromCrash('Worker threw an error');
  }

  private handleUnresponsive(): void {
    console.warn('Worker unresponsive, terminating');
    this.recoverFromCrash('Worker became unresponsive');
  }

  private recoverFromCrash(reason: string): void {
    // 1. Terminate the dead worker
    this.worker?.terminate();
    this.worker = null;

    // 2. Notify UI to show recovery dialog
    this.onCrash?.({
      reason,
      options: ['loadLastAutosave', 'loadManualSave', 'startNewGame'],
    });

    // 3. After user chooses, call start() again with the selected save
  }

  onCrash?: (info: { reason: string; options: string[] }) => void;
}
```

### Corrupted Save Recovery

Saves can become corrupted due to interrupted writes or bugs. Implement validation and fallback:

```ts
import { z } from 'zod'; // or use custom validators

// Define schema for runtime validation
const SimStateSchema = z.object({
  version: z.number(),
  tick: z.number().nonnegative(),
  day: z.number().nonnegative(),
  rng: z.object({ seed: z.number(), state: z.array(z.number()) }),
  player: z.object({
    money: z.number(),
    reputation: z.number().min(0).max(100),
    compute: z.object({ owned: z.number(), allocated: z.number() }),
  }),
  // ... other fields
});

export function validateSave(save: unknown): { valid: true; save: SaveFile } | { valid: false; errors: string[] } {
  try {
    // 1. Check basic structure
    if (!save || typeof save !== 'object') {
      return { valid: false, errors: ['Save is not an object'] };
    }

    // 2. Check version and migrate if needed
    const raw = save as Record<string, unknown>;
    if (typeof raw.saveVersion !== 'number') {
      return { valid: false, errors: ['Missing saveVersion'] };
    }

    // 3. Validate sim state
    const result = SimStateSchema.safeParse(raw.sim);
    if (!result.success) {
      return { valid: false, errors: result.error.errors.map((e) => e.message) };
    }

    // 4. Check invariants
    const invariantErrors = checkInvariants(result.data);
    if (invariantErrors.length > 0) {
      return { valid: false, errors: invariantErrors };
    }

    return { valid: true, save: save as SaveFile };
  } catch (e) {
    return { valid: false, errors: [`Validation threw: ${e}`] };
  }
}

function checkInvariants(sim: SimState): string[] {
  const errors: string[] = [];
  
  // Example invariants
  if (sim.player.compute.allocated > sim.player.compute.owned) {
    errors.push('Allocated compute exceeds owned compute');
  }
  if (sim.day < 0 || sim.tick < 0) {
    errors.push('Negative time values');
  }
  
  return errors;
}
```

Recovery flow for corrupted saves:
1. Attempt to load the requested save.
2. If validation fails, try the most recent autosave for that slot.
3. If all autosaves fail, offer to start a new game or import a backup.
4. Log the corruption for debugging (include save metadata, not full state).

### Graceful Degradation

```ts
export type RecoveryAction =
  | { type: 'loadedAutosave'; slotId: string; autosaveIndex: number }
  | { type: 'startedNewGame'; reason: string }
  | { type: 'userChoseManualSave'; slotId: string };

// Show non-blocking notification for minor issues
export function notifyRecovery(action: RecoveryAction): void {
  // Display toast: "Loaded autosave due to corrupted save file"
}
```

## Replay / Debug Mode

### Why
- Verifiable bug reports
- Balancing regressions detection
- AI-assisted debugging: replay is a single artifact

### Replay file
Record deterministic inputs:

```ts
export type ReplayFile = {
  replayVersion: number;
  seed: number;
  initial: NewGameConfig | SaveFile;
  commands: Array<{ id: string; atTick: number; command: GameCommand }>;
};
```

Optional enhancements:
- Add periodic checkpoints to allow fast-forward.
- Allow exporting/importing replay JSON from a debug menu.

### Headless runner
Add a dev-only way to run the sim from a replay without the UI.
- Used by unit tests and regression checks.

## Codebase Conventions (AI + Human Friendly)
- TypeScript everywhere.
- Prefer explicit data over implicit behavior.
- Keep modules small and single-purpose.
- No cross-layer imports:
  - `ui/` must not import `worker/` internals
  - `sim/` must not import `ui/`
- Centralize schemas/types in `shared/`.
- Keep functions pure in `sim/`.

## Suggested Tech Stack
- Build: Vite + React + TypeScript
- Testing: Vitest (sim core unit tests + replay regression tests)

## Delivery Roadmap (phased)

Each phase includes: file structure, deliverables, acceptance criteria, and test expectations.

---

### Phase 1: Thin Vertical Slice

**Goal**: Prove the Worker ↔ UI architecture with minimal game mechanics.

#### 1.1 File Structure

```
src/
├── shared/
│   ├── types/
│   │   ├── sim-state.ts      # SimState, Player, Job types
│   │   ├── commands.ts       # GameCommand union
│   │   ├── messages.ts       # ClientMessage, WorkerMessage
│   │   └── view-snapshot.ts  # ViewSnapshot, JobView
│   ├── rng.ts                # Seeded PRNG implementation
│   └── config.ts             # TICKS_PER_DAY, DAYS_PER_MONTH, etc.
├── sim/
│   ├── state/
│   │   ├── initial.ts        # createInitialState(seed, config)
│   │   └── invariants.ts     # checkInvariants(state): string[]
│   ├── commands/
│   │   ├── validate.ts       # validateCommand(state, cmd): Result
│   │   └── apply.ts          # applyCommand(state, cmd): state
│   ├── systems/
│   │   ├── finance.ts        # runFinance(state, ctx)
│   │   └── training.ts       # runTraining(state, ctx)
│   └── engine/
│       ├── tick.ts           # advanceTick(state, ctx): { state, events }
│       └── scheduler.ts      # ScheduledEventQueue helpers
├── worker/
│   ├── sim.worker.ts         # Worker entry, message handler, tick loop
│   ├── loop.ts               # Tick loop with accumulator, speed control
│   └── snapshot.ts           # buildViewSnapshot(state): ViewSnapshot
├── ui/
│   ├── store/
│   │   ├── game-store.ts     # Zustand store for ViewSnapshot
│   │   └── worker-bridge.ts  # postMessage helpers, WorkerManager
│   ├── components/
│   │   ├── Dashboard.tsx     # Main layout
│   │   ├── KpiBar.tsx        # Money, compute, day display
│   │   ├── JobList.tsx       # List of training jobs
│   │   ├── SpeedControls.tsx # Pause, 1x, 2x, 5x buttons
│   │   └── SaveLoadPanel.tsx # Single slot save/load
│   └── App.tsx               # Root component
├── persistence/
│   ├── db.ts                 # IndexedDB setup (idb wrapper)
│   └── save-load.ts          # save(slot, state), load(slot)
└── main.tsx                  # React entry point
```

#### 1.2 Deliverables (in order)

| # | Task | Output |
|---|------|--------|
| 1 | Seeded PRNG | `shared/rng.ts` with `createRng(seed)`, `rng.next()`, `rng.nextInt(min, max)` |
| 2 | Core types | `SimState`, `Player`, `Job`, `GameCommand`, `ViewSnapshot` |
| 3 | Initial state factory | `createInitialState(seed, config)` returns valid `SimState` |
| 4 | Command validation | `validateCommand(state, cmd)` returns `{ valid: true }` or `{ valid: false, reason }` |
| 5 | Command application | `applyCommand(state, cmd)` returns new state (immutable) |
| 6 | Finance system | Deduct daily compute cost, no revenue yet |
| 7 | Training system | Progress jobs by `1 / estimatedDays` per day, complete when progress >= 1 |
| 8 | Tick engine | `advanceTick` calls systems in order, handles day boundaries |
| 9 | Worker tick loop | Accumulator-based loop, respects `speed`, emits `ViewSnapshot` |
| 10 | Worker message handler | Handle `command`, `save`, `load`, `requestView` |
| 11 | Snapshot builder | `buildViewSnapshot(state)` with player stats + job list |
| 12 | Zustand store | Store `ViewSnapshot`, expose selectors |
| 13 | WorkerBridge | `sendCommand`, `requestSave`, `requestLoad` |
| 14 | UI: KpiBar | Display money, compute owned/allocated, current day |
| 15 | UI: JobList | List jobs with progress bars |
| 16 | UI: SpeedControls | Pause/play toggle, speed buttons |
| 17 | UI: SaveLoadPanel | Single slot save/load buttons |
| 18 | IndexedDB persistence | `saveGame(slotId, state)`, `loadGame(slotId)` |
| 19 | Pause on tab hidden | `visibilitychange` listener sends `SetPaused(true)` |

#### 1.3 Acceptance Criteria

- [ ] Game starts with $100,000, 10 compute units, day 1
- [ ] One hardcoded training job ("Foundation Model v1") starts automatically
- [ ] Job progresses visibly (~10 days to complete at 1x speed)
- [ ] Pause stops all progression; resume continues from same tick
- [ ] Speed 2x/5x makes job progress proportionally faster
- [ ] Save persists to IndexedDB; reload page + load restores exact state
- [ ] Tab hidden → game pauses; tab visible → remains paused until user resumes
- [ ] No `Math.random()` calls in `sim/` or `shared/`

#### 1.4 Test Expectations

```ts
// sim/engine/tick.test.ts
describe('advanceTick', () => {
  it('advances tick counter by 1', () => { ... });
  it('increments day when tick % TICKS_PER_DAY === 0', () => { ... });
  it('produces identical state given same seed and commands', () => { ... });
});

// sim/systems/training.test.ts
describe('runTraining', () => {
  it('increments job progress on day boundary', () => { ... });
  it('marks job complete when progress >= 1', () => { ... });
  it('does not modify jobs when not on day boundary', () => { ... });
});

// sim/commands/apply.test.ts
describe('applyCommand', () => {
  it('SetSpeed updates speed without affecting tick', () => { ... });
  it('SetPaused(true) sets isPaused flag', () => { ... });
});

// worker/snapshot.test.ts
describe('buildViewSnapshot', () => {
  it('includes all jobs sorted by progress descending', () => { ... });
  it('computes computeAvailable correctly', () => { ... });
});
```

---

### Phase 2: Core Tycoon Loop

**Goal**: Playable game loop with staff, models, products, and revenue.

#### 2.1 New Files

```
src/
├── shared/types/
│   ├── staff.ts              # Staff, StaffRole
│   ├── model.ts              # Model, ModelQuality
│   ├── product.ts            # Product, ProductPricing
│   ├── market.ts             # MarketSegment, Demand
│   └── competitor.ts         # Competitor, CompetitorPersonality
├── sim/
│   ├── systems/
│   │   ├── payroll.ts        # Deduct salaries daily
│   │   ├── market.ts         # Update demand, calculate revenue
│   │   └── competitor.ts     # Basic competitor decisions
│   └── commands/
│       └── handlers/
│           ├── hire-staff.ts
│           ├── start-training.ts
│           ├── launch-product.ts
│           └── set-pricing.ts
├── ui/components/
│   ├── StaffPanel.tsx        # Hire/fire, assign to jobs
│   ├── ModelList.tsx         # Completed models
│   ├── ProductList.tsx       # Active products with revenue
│   ├── MarketOverview.tsx    # Segment demand bars
│   └── SaveSlotPicker.tsx    # Multiple save slots
└── persistence/
    └── autosave.ts           # Autosave timer, rolling slots
```

#### 2.2 Deliverables (in order)

| # | Task | Output |
|---|------|--------|
| 1 | Staff types + state | `Staff`, `StaffRole`, add `staffById` to `SimState` |
| 2 | HireStaff command | Validate budget, create staff entity |
| 3 | Payroll system | Deduct `sum(staff.salary)` daily |
| 4 | Staff assignment | Assign staff to jobs, affects training speed |
| 5 | Model types + state | `Model`, `ModelQuality`, add `modelsById` |
| 6 | Training completion | Creates `Model` entity when job completes |
| 7 | Product types + state | `Product`, add `productsById` |
| 8 | LaunchProduct command | Create product from model + market segment |
| 9 | Market types + state | `MarketSegment`, `Demand`, add `market` to state |
| 10 | Revenue system | Calculate daily revenue from products × demand × pricing |
| 11 | SetPricing command | Adjust product price, affects demand |
| 12 | Competitor types | `Competitor`, `CompetitorPersonality` |
| 13 | Basic competitor AI | Competitors adjust prices daily (simple rules) |
| 14 | UI: StaffPanel | List staff, hire button (2-3 predefined roles) |
| 15 | UI: ModelList | Show completed models |
| 16 | UI: ProductList | Show products with revenue/day |
| 17 | UI: MarketOverview | Show segments with demand levels |
| 18 | Multiple save slots | UI to pick from 5 slots |
| 19 | Autosave | Every 60 seconds, keep last 3 autosaves per slot |

#### 2.3 Acceptance Criteria

- [ ] Can hire 3 staff roles: Researcher ($500/day), Engineer ($400/day), Ops ($300/day)
- [ ] Staff assigned to job increases training speed (e.g., +20% per staff)
- [ ] Completed training creates a Model with quality based on compute + staff
- [ ] Can launch product into 1 of 3 market segments (Consumer, Enterprise, API)
- [ ] Products generate revenue = `demand × price × modelQuality`
- [ ] Lowering price increases demand (simple elasticity)
- [ ] 2 AI competitors exist, adjust prices weekly
- [ ] Autosave triggers every 60 seconds
- [ ] Can save to and load from 5 different slots

#### 2.4 Test Expectations

```ts
// sim/systems/payroll.test.ts
describe('runPayroll', () => {
  it('deducts total salaries on day boundary', () => { ... });
  it('does nothing if no staff', () => { ... });
});

// sim/systems/market.test.ts
describe('runMarket', () => {
  it('calculates revenue correctly', () => { ... });
  it('adjusts demand based on price changes', () => { ... });
});

// sim/commands/handlers/hire-staff.test.ts
describe('HireStaff', () => {
  it('rejects if insufficient funds', () => { ... });
  it('creates staff entity with correct salary', () => { ... });
});
```

---

### Phase 3: Depth + Balancing

**Goal**: Strategic depth, competitor variety, historical trends, replay support.

#### 3.1 New Files

```
src/
├── shared/types/
│   ├── research.ts           # ResearchProject, TechTree node
│   ├── upgrade.ts            # ComputeUpgrade, Unlockable
│   └── events.ts             # GameEvent union for logging
├── sim/
│   ├── systems/
│   │   ├── research.ts       # Unlock new capabilities
│   │   └── events.ts         # Emit GameEvents for history
│   ├── history.ts            # Append-only event log for trends
│   └── replay/
│       ├── recorder.ts       # Record commands with tick
│       └── runner.ts         # Headless replay runner
├── ui/components/
│   ├── Charts/
│   │   ├── RevenueChart.tsx
│   │   ├── ReputationChart.tsx
│   │   └── MarketShareChart.tsx
│   ├── TechTree.tsx          # Research unlock visualization
│   ├── CompetitorDetail.tsx  # Expanded competitor view
│   ├── EventLog.tsx          # Recent game events
│   └── DebugPanel.tsx        # Replay export, state inspector
└── tests/
    └── replay/
        ├── fixtures/         # Saved replay JSON files
        └── regression.test.ts
```

#### 3.2 Deliverables (in order)

| # | Task | Output |
|---|------|--------|
| 1 | GameEvent types | Union type for all loggable events |
| 2 | Event emission | Systems emit events (job complete, product launched, etc.) |
| 3 | Event history | `state.eventLog: GameEvent[]` (capped at last 100) |
| 4 | Research types | `ResearchProject`, unlocks (e.g., "Larger Models", "Efficient Training") |
| 5 | Research system | Progress research, unlock capabilities |
| 6 | Compute upgrades | Buy more compute, unlock tiers |
| 7 | Model types | 3 model sizes (Small, Medium, Large) with different costs/quality |
| 8 | Competitor strategies | Full strategy system from spec (expand, defend, undercut, etc.) |
| 9 | Competitor personalities | Vary aggressiveness, innovation focus, etc. |
| 10 | ViewSnapshot charts | Add `charts` field with daily data points |
| 11 | UI: RevenueChart | Line chart of last 30 days |
| 12 | UI: ReputationChart | Line chart with competitor comparison |
| 13 | UI: MarketShareChart | Stacked area chart |
| 14 | UI: TechTree | Visual unlock tree |
| 15 | UI: EventLog | Scrollable list of recent events |
| 16 | Replay recorder | `CommandRecorder` captures commands with tick |
| 17 | Replay export | Debug menu to download replay JSON |
| 18 | Replay import | Load replay JSON, run headlessly or in UI |
| 19 | Headless runner | `runReplay(replay): FinalState` for tests |
| 20 | Regression tests | Replay fixtures that assert final state |

#### 3.3 Acceptance Criteria

- [ ] 5+ research unlocks available (e.g., training efficiency, model size, market access)
- [ ] Compute can be upgraded (Tier 1: 10 units → Tier 2: 50 units → Tier 3: 200 units)
- [ ] 3 model sizes with cost/quality tradeoffs
- [ ] Competitors switch strategies based on market position
- [ ] Charts show 30-day history, update daily
- [ ] Event log shows last 20 events with timestamps
- [ ] Replay export produces valid JSON
- [ ] Replay import recreates identical game state
- [ ] At least 3 replay regression tests pass

#### 3.4 Test Expectations

```ts
// sim/replay/runner.test.ts
describe('runReplay', () => {
  it('produces identical final state for fixture-1.json', () => { ... });
  it('produces identical final state for fixture-2.json', () => { ... });
});

// sim/systems/competitor.test.ts
describe('runCompetitorSystem', () => {
  it('switches to undercut strategy when player grows fast', () => { ... });
  it('respects strategyMinLockDays', () => { ... });
});

// sim/systems/research.test.ts
describe('runResearch', () => {
  it('unlocks capability when research completes', () => { ... });
});
```

---

### Phase 4: Optimization & Polish

**Goal**: Production-ready performance and UX.

#### 4.1 New/Modified Files

```
src/
├── worker/
│   └── snapshot-diff.ts      # Delta patches instead of full snapshots
├── ui/
│   ├── components/
│   │   ├── VirtualList.tsx   # Generic virtualized list
│   │   ├── Tooltips.tsx      # Contextual help
│   │   └── FilterBar.tsx     # Filter jobs/staff/products
│   ├── hooks/
│   │   └── useVirtualList.ts
│   └── perf/
│       └── profiler.tsx      # React Profiler wrapper
├── persistence/
│   └── compression.ts        # Compress saves (optional)
└── tests/
    └── perf/
        ├── snapshot-size.test.ts
        └── tick-perf.test.ts
```

#### 4.2 Deliverables

| # | Task | Output |
|---|------|--------|
| 1 | Profiling baseline | Measure tick time, snapshot size, render time |
| 2 | Snapshot diffing | Only send changed fields (if snapshot > 50KB) |
| 3 | Virtualized lists | JobList, StaffPanel virtualized for 100+ items |
| 4 | Memoization audit | Ensure selectors and components are memoized |
| 5 | Filter UI | Filter jobs by status, staff by role, products by segment |
| 6 | Tooltips | Hover help for all KPIs and buttons |
| 7 | Keyboard shortcuts | Space = pause, 1/2/3 = speed, S = save |
| 8 | Save compression | Optional LZ-string compression for large saves |
| 9 | Error boundaries | Graceful fallback for UI crashes |
| 10 | Loading states | Skeleton UI while worker initializes |

#### 4.3 Acceptance Criteria

- [ ] Tick loop runs < 5ms for 200 entities
- [ ] ViewSnapshot < 100KB (or delta < 5KB)
- [ ] UI renders 200 jobs at 60fps with virtualization
- [ ] All interactive elements have tooltips
- [ ] Keyboard shortcuts work globally
- [ ] Save/load works with 1MB+ game states
- [ ] No unhandled React errors in production build

#### 4.4 Test Expectations

```ts
// tests/perf/tick-perf.test.ts
describe('tick performance', () => {
  it('completes tick in < 5ms with 200 entities', () => {
    const state = createLargeState({ jobs: 100, staff: 50, products: 50 });
    const start = performance.now();
    advanceTick(state, ctx);
    expect(performance.now() - start).toBeLessThan(5);
  });
});

// tests/perf/snapshot-size.test.ts
describe('snapshot size', () => {
  it('full snapshot < 100KB for large state', () => {
    const snapshot = buildViewSnapshot(largeState);
    const size = JSON.stringify(snapshot).length;
    expect(size).toBeLessThan(100_000);
  });
});
```

---

## Implementation Notes for AI Agents

1. **Follow the order**: Deliverables are numbered for dependency order. Complete each before starting the next.

2. **Type-first**: Define types in `shared/types/` before implementing logic.

3. **Test as you go**: Write tests for each deliverable before marking complete.

4. **Check acceptance criteria**: After completing a phase, verify all criteria before proceeding.

5. **Commit granularly**: One commit per deliverable with message format: `phase1.3: add initial state factory`

6. **Ask for clarification**: If a deliverable is ambiguous, you are always very welcome to ask before implementing.
