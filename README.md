<p align="center">
  <h1 align="center">VRP-RPD Solver</h1>
  <p align="center">Route optimization for Indian logistics — delivery fleets with resource-constrained pickup and delivery.</p>
  <p align="center">
    <a href="#installation"><img src="https://img.shields.io/badge/node-20%2B-brightgreen" alt="Node"></a>
    <a href="LICENSE"><img src="https://img.shields.io/badge/license-ISC-green" alt="License"></a>
    <a href="https://github.com/sachncs/vehicle-routing-problem-with-resource-constraints/actions"><img src="https://img.shields.io/github/actions/workflow/status/sachncs/vehicle-routing-problem-with-resource-constraints/ci.yml?branch=master" alt="CI"></a>
    <a href="https://www.npmjs.com/package/vehicle-routing"><img src="https://img.shields.io/npm/v/vehicle-routing" alt="npm"></a>
    <a href="https://github.com/sachncs/vehicle-routing-problem-with-resource-constraints/stargazers"><img src="https://img.shields.io/github/stars/sachncs/vehicle-routing-problem-with-resource-constraints" alt="Stars"></a>
  </p>
</p>

**Vehicle Routing Problem with Resource-Constrained Pickup and Delivery (VRP-RPD) — a two-stage metaheuristic (ALNS + BRKGA) solver for Indian logistics fleets.**

This library solves VRP-RPD where goods must be delivered, processed, and then picked up, potentially by different vehicles across multiple trips. It uses a two-stage metaheuristic — Adaptive Large Neighborhood Search followed by Biased Random-Key Genetic Algorithm — to find high-quality routes fast.

**Built for Indian logistics:** Supports time windows, multi-depot operations, traffic-aware routing, inter-vehicle transfers at hub nodes, multi-objective optimization (cost, distance, CO₂), and island-model parallel BRKGA via `worker_threads`.

## Algorithmic Improvements Beyond the Paper

This implementation surpasses the baseline algorithms described in arXiv:2602.23685v2 with several novel enhancements:

| Improvement | Description |
|-------------|-------------|
| **Adaptive Removal Sizing** | ALNS removal fraction auto-adjusts 10% → 45% based on stagnation ratio |
| **Multi-Restart ALNS** | Up to 3 restarts with temperature reset and weight zeroing on stagnation |
| **Clone Avoidance** | ALNS only clones solution on new best, avoiding regressions |
| **Elite Diversity Preservation** | Mild mutation on elite BRKGA copies proportional to stagnation |
| **Adaptive Mutation Rate** | Up to +5% extra mutants injected when population stagnates |
| **Immigrant Injection** | 20% of population replaced with fresh random individuals before breaking stagnation |
| **Hall-of-Fame Tracking** | Best-ever solution tracked separately from population elite |
| **Decoder O(1) Capacity Checks** | Incremental `RouteLoad` tracking replaces O(n) route simulation |
| **Island-Model Parallelization** | Multi-population BRKGA with elite migration via `worker_threads` |

## Features

- **ALNS** — Adaptive Large Neighborhood Search — 6 destroy + 4 repair operators, adaptive weight selection
- **BRKGA** — Biased Random-Key Genetic Algorithm — 4n chromosome, elite / mutant / crossover evolution
- **Warm-start** — ALNS solution seeds 15% of BRKGA population for faster convergence
- **Time windows** — Earliest / latest delivery and pickup constraints (VRPTW)
- **Multi-depot** — Vehicles can start / end at different depots
- **Traffic-aware** — Time-dependent travel speeds via traffic model
- **Inter-vehicle transfers** — Exchange resources at hub nodes
- **Multi-objective** — Pareto optimization for makespan, distance, cost, CO₂
- **Analytics** — Vehicle utilization, wait times, load profiles, route comparison
- **GIS export** — GeoJSON, KML, CSV output for QGIS, Google Earth, Excel
- **Serialization** — Save / load solutions as JSON
- **Parallel solving** — Run ALNS and BRKGA concurrently via worker threads
- **Progress callback** — Real-time progress with iteration and best makespan

## Installation

### From npm

```bash
npm install vehicle-routing
```

### From source

```bash
git clone https://github.com/sachncs/vehicle-routing-problem-with-resource-constraints.git
cd vehicle-routing-problem-with-resource-constraints
npm install
```

## Quick Start

### CLI

```bash
# Install globally to use the vrp-solver binary
npm install -g vehicle-routing

vrp-solver --problem samples/delhi-10.json --output solution.json
vrp-solver --problem samples/mumbai-20.json --progress
```

| Option | Default | Description |
|--------|---------|-------------|
| `--problem <file>` | required | Path to problem JSON file |
| `--output <file>` | stdout | Write solution JSON |
| `--alns-iterations <n>` | `500` | ALNS iterations |
| `--population-size <n>` | `30000` | BRKGA population size |
| `--max-generations <n>` | `20000` | BRKGA max generations |
| `--max-time <ms>` | `0` (unlimited) | Max solver time |
| `--target-makespan <n>` | `0` (disabled) | Early stopping target |
| `--parallel` | off | Run ALNS + BRKGA in parallel |
| `--no-warm-start` | on | Disable ALNS warm-start |
| `--progress` | off | Print progress to stderr |

### Node.js API (TypeScript)

```typescript
import { VrpRpdSolver, VrpProblem, LocationNode, Customer, Vehicle } from 'vehicle-routing';

const nodes = {
  0: new LocationNode(0, 0, 0, 'Depot'),
  1: new LocationNode(1, 10, 0, 'Customer A - Drop'),
  2: new LocationNode(2, 20, 0, 'Customer A - Pick'),
};
const customers = [new Customer(1, 1, 2, 50)]; // id, del-node, pk-node, processing-minutes
const vehicles  = [new Vehicle(1, 5)];          // id, capacity

const problem = new VrpProblem(nodes, customers, vehicles, 0);
const solver  = new VrpRpdSolver(problem);

const solution = await solver.solve({ maxTimeMs: 30000 });
console.log(`Best makespan: ${solution.makespan.toFixed(2)} min`);
console.log(`Feasible: ${solution.isFeasible()}`);
console.log(`Distance: ${solution.totalDistance.toFixed(2)} km`);
```

## Solver Options

```typescript
interface SolveOptions {
  alnsIterations?: number;       // Default: 500
  populationSize?: number;       // Default: 30000
  maxGenerations?: number;       // Default: 20000
  initialTemp?: number;          // Default: 100
  coolingRate?: number;          // Default: 0.9998
  parallel?: boolean;            // Default: false
  warmStart?: boolean;           // Default: true
  maxTimeMs?: number;            // Default: 0 (unlimited)
  targetMakespan?: number;       // Default: 0 (disabled)
  islands?: number;              // Default: 1
  migrationInterval?: number;    // Default: 50
  migrantFraction?: number;      // Default: 0.05
  logger?: Logger;
  onProgress?: (p: SolverProgress) => void;
}
```

Island-model BRKGA (multi-core):

```typescript
const solution = await solver.solve({
  islands: 4,              // 4 parallel populations
  migrationInterval: 50,   // exchange elites every 50 generations
  populationSize: 30000,
  maxGenerations: 20000,
});
```

## Configuration

No environment variables are required for core usage. Defaults are tuned for paper-quality results.

| Setting | Default | Description |
|---------|---------|-------------|
| `alnsIterations` | `500` | ALNS iteration cap |
| `populationSize` | `30000` | BRKGA population size |
| `maxGenerations` | `20000` | BRKGA generation cap |
| `initialTemp` | `100` | Simulated-annealing start temperature |
| `coolingRate` | `0.9998` | Geometric cooling factor |
| `parallel` | `false` | Run ALNS + BRKGA concurrently |
| `warmStart` | `true` | Seed 15% of BRKGA from ALNS solution |
| `maxTimeMs` | `0` | Wall-clock cap (`0` = unlimited) |
| `targetMakespan` | `0` | Early-stop on reaching target |
| `islands` | `1` | BRKGA island count |
| `migrationInterval` | `50` | Generations between elite migrations |
| `migrantFraction` | `0.05` | Fraction of migrants per migration |

## API

| Symbol | Type | Description |
|--------|------|-------------|
| `VrpRpdSolver` | class | Orchestrator (ALNS → warm-start → BRKGA) |
| `VrpProblem` | class | Standard problem definition |
| `TrafficAwareProblem` | class | Problem with time-dependent travel times |
| `MultiDepotProblem` | class | Multi-depot variant |
| `Customer` / `CustomerWithTimeWindows` | class | Customer with optional delivery / pickup windows |
| `Vehicle` / `VehicleWithCapabilities` | class | Vehicle with capacity and directional capabilities |
| `LocationNode` | class | Node with coordinates |
| `RouteAnalytics` | class | Post-solution summary metrics |
| `SolutionComparator` | class | Pareto-front comparison |
| `GISExporter` | class | `.toGeoJSON()` / `.toKML()` / `.toCSV()` |
| `TrafficModel` | class | Time-dependent segment factors |
| `VrpError` → `ValidationError` \| `InfeasibleSolutionError` \| `AlgorithmConvergenceError` | classes | Typed error hierarchy |

## Examples

### With Time Windows

```typescript
import { CustomerWithTimeWindows } from 'vehicle-routing';

// Deliver between 9 AM and 1 PM (360–480 min), pick up between 11 AM and 10 PM (420–600 min)
const customer = new CustomerWithTimeWindows(
  1, 1, 2,              // id, delivery node, pickup node
  30,                    // 30-min processing
  360, 480,              // earliest / latest delivery
  420, 600,              // earliest / latest pickup
);
```

### Traffic-Aware Routing

```typescript
import { TrafficAwareProblem, TrafficModel } from 'vehicle-routing';

const traffic = new TrafficModel();
traffic.addSegment(depotNode, customerNode, {
  baseTravelTime: 30,
  timeDependentFactors: { 8: 1.5, 9: 2.0, 17: 1.8, 18: 1.6 }, // rush hour
  congestionLevel: 1.5,
});

const problem = new TrafficAwareProblem(nodes, customers, vehicles, 0, traffic);
```

### Analytics & GIS Export

```typescript
import { RouteAnalytics, GISExporter } from 'vehicle-routing';

const analytics = new RouteAnalytics(solution, problem);
console.log(analytics.getSummary());
// { makespan, totalDistance, totalCost, totalCO2, avgUtilization, ... }

const exporter = new GISExporter(solution, problem);
const geojson = exporter.toGeoJSON();   // QGIS / Mapbox
const kml     = exporter.toKML();       // Google Earth
const csv     = exporter.toCSV();       // Excel
```

### Progress Tracking

```typescript
const solution = await solver.solve({
  onProgress: (p) => {
    console.log(`[${p.stage}] ${p.iteration}/${p.maxIterations} — best: ${p.bestMakespan.toFixed(1)}min`);
  },
});
```

### Problem JSON Format

```json
{
  "nodes": [
    { "id": 0, "x": 28.61, "y": 77.23, "name": "Delhi Depot" },
    { "id": 1, "x": 28.54, "y": 77.20, "name": "Customer 1 Drop" },
    { "id": 2, "x": 28.56, "y": 77.25, "name": "Customer 1 Pick" }
  ],
  "customers": [
    { "id": 1, "deliveryNodeId": 1, "pickupNodeId": 2, "processingTime": 30 }
  ],
  "vehicles": [
    { "id": 1, "capacity": 100, "costPerKm": 12, "co2PerKm": 0.15 }
  ],
  "depotNodeId": 0
}
```

For time windows, add these fields to customers: `earliestDeliveryTime`, `latestDeliveryTime`, `earliestPickupTime`, `latestPickupTime`.

## Architecture

The solver is organized into four layers:

| Layer | Directory | Responsibility |
|-------|-----------|----------------|
| **Core** | `src/core/` | Problem definition, solution model, schedule engine |
| **Algorithms** | `src/algorithms/` | ALNS metaheuristic, BRKGA evolutionary algorithm, chromosome decoder |
| **Analytics** | `src/analytics/` | Post-solution analysis, Pareto front computation |
| **Export** | `src/export/` | GIS serialization (GeoJSON, KML, CSV) |

The `VrpRpdSolver` orchestrator runs a **two-stage metaheuristic**: ALNS first, then BRKGA warm-started from the ALNS solution. In parallel mode both run concurrently via `worker_threads` and the best result is returned.

```
Problem ──► ALNS (adaptive destroy/repair) ──► warm-start ──► BRKGA (evolutionary) ──► Best Solution
                                                     │
                                               (chromosome encoding
                                                of ALNS solution,
                                                15% of initial pop)
```

## Design Philosophy

- **Strategy pattern** — ALNS destroy / repair operators are interchangeable functions selected via weighted roulette, enabling easy addition of new operators.
- **Adaptive weighting** — Operator weights update every segment via reinforcement learning: `w[i] = (1-λ)·w[i] + λ·(score[i]/usage[i])` with λ = 0.1.
- **Simulated annealing acceptance** — Worsening solutions accepted with probability `exp((cost_current - cost_new) / temperature)`; temperature decays geometrically.
- **Biased crossover (BRKGA)** — Each child inherits each gene from the elite parent with probability 0.7.
- **Message-passing IPC** — Worker threads communicate via a typed request-response protocol supporting `evolve`, `inject`, and `finish` commands.
- **Logger DI** — A minimal `Logger` interface allows silent library use or custom logging without coupling to a framework.
- **Error hierarchy** — `VrpError` → `ValidationError` | `InfeasibleSolutionError` | `AlgorithmConvergenceError`.

## Key Implementation Details

### O(1) Capacity Checks via Incremental `RouteLoad`

The decoder tracks three values per route: `RouteLoad { currentLoad, minDelta, maxDelta }`. A new operation is feasible iff `initialLoadNeeded ≤ capacity` and `peakLoad ≤ capacity`, where `initialLoadNeeded = -newMin` and `peakLoad = initialLoadNeeded + newMax`. This avoids an O(n) scan of the entire route on every insertion.

### Adaptive Removal Sizing

`removalFraction = 0.1 + stagnationRatio × 0.35`, growing from 10% to 45% as stagnation increases.

### Multi-Restart ALNS

When `iterationsSinceImprovement ≥ maxStagnation` (5% of max iterations, min 25), ALNS restarts: temperature resets to `initialTemp × 0.5^restartsUsed`, cooling rate becomes `baseCooling × (1 + restartsUsed × 0.02)`, all operator weights zero, up to 3 restarts allowed.

### Two-Pass Greedy Decoder

The 4n-gene chromosome is decoded in three passes: (1) deliveries scheduled in priority order, assigned via `σ` genes; (2) delivery visit times computed (needed for pickup feasibility); (3) pickups scheduled respecting resource-ready-time constraints.

### Chromosome Structure (4n genes, arXiv:2602.23685v2)

```
Chromosome = { priorities (π), assignments (σ), dependencies (α), transfers (β) }
```

Each gene is a float in `[0, 1)`. Total size = 4 × numCustomers.

### Warm-Start Encoding

`priorities[i] = (routeIdx × 100 + position) / (numRoutes × 100)`, `assignments[i] = routeIdx / numVehicles` — seeds 15% of the initial BRKGA population.

### Schedule Fixed-Point Iteration

Cross-route resource dependencies require iterative schedule computation. The loop visits all routes repeatedly until node times converge (max 1000 iterations), propagating resource-ready-time updates between vehicles.

### Stagnation Resistance in BRKGA

- **Elite diversity preservation**: mild mutation on elite copies (`rate = min(0.05, stagnationRatio × 0.1)`).
- **Adaptive mutation rate**: extra mutants = `⌊stagnationRatio × populationSize × 0.05⌋`.
- **Immigrant injection**: 20% of population replaced with fresh random individuals before breaking stagnation.

## The Math

### Problem Formulation (VRP-RPD)

Each customer `c` has a delivery node `D_c`, a pickup node `P_c`, and a processing time `p_c`. The resource constraint is:

```
arrivalTime(P_c) ≥ arrivalTime(D_c) + p_c
```

This creates temporal dependencies: one vehicle may deliver, another may pick up, but pickup cannot start until processing completes.

### Distance and Travel Time

Euclidean distance between nodes: `distance(i, j) = √((x_i - x_j)² + (y_i - y_j)²)`. Travel time: `distance / speed`, optionally modified by the traffic model.

### ALNS — Adaptive Weights

Every `segmentSize` iterations (default 50): `w[i] = (1 - λ) · w[i] + λ · (score[i] / usage[i])`. Score per operator: new global best +33, better than current +9, accepted via SA +13, rejected 0. Selection probability: `P(i) = w[i] / Σⱼ w[j]`.

### ALNS — SA Acceptance Probability

`P(accept worse) = exp((cost_current - cost_new) / temperature)`; `temp ← temp × coolingRate` each iteration.

### BRKGA — Biased Crossover

Each gene of a child: `child[i] = elite[i]` with probability 0.7, else `nonElite[i]`.

### BRKGA — Evolution per Generation

1. Sort population by fitness ascending
2. Copy top `e × popSize` elites (with mild mutation)
3. Replace bottom `m × popSize` with random mutants
4. Fill remainder via biased crossover between random elite + random non-elite

### Regret Insertion (ALNS Repair)

`regret_k(c) = cost_k(c) - cost_1(c)`. The customer with the largest regret (most "urgent") is inserted first. If fewer than `k` routes can accommodate a customer, a fallback cost is used.

### Shaw Relatedness (ALNS Destroy)

`relatedness(c₁, c₂) = ||D_c₁ - D_c₂||₂ + |time(D_c₁) - time(D_c₂)|`. Lower relatedness = more similar = removed together.

### Multi-Objective Pareto Front

A solution is Pareto-optimal if no other solution dominates it. Dominance: `other dominates current` iff all objectives ≤ current **and** strictly better in at least one of `{makespan, totalDistance, totalCost, totalCO₂}`.

### Traffic Model

Time-dependent travel speed: if `departureTime ≥ latest-matching factor.startTime`, `travelTime = baseTravelTime × factor.multiplier`; else `travelTime = segment.currentTravelTime`. Congestion level from `newTravelTime / baseTravelTime` (low < 1.2, medium < 1.5, high < 2.0, severe otherwise).

### Resource Transfers

`transferDuration = amount × hub.transferTimePerUnit`. Each hub enforces a concurrency limit. Vehicles have directional transfer permissions.

## Performance Tips

- **Quick test:** `alnsIterations: 100, populationSize: 1000, maxGenerations: 500`
- **Production:** `alnsIterations: 500, populationSize: 30000, maxGenerations: 20000` (paper defaults)
- **Time-constrained:** Set `maxTimeMs` to stop early with a feasible solution
- **Multi-core:** Enable `parallel: true` (ALNS + BRKGA concurrently) or `islands: 4` (parallel BRKGA populations with elite migration)
- **Stagnation resistance:** The ALNS multi-restart and BRKGA adaptive-mutation / immigrant-injection mechanisms automatically handle most convergence issues

## Development

```bash
npm install
npm run build              # rollup ESM + CJS bundles
npm run dev                # rollup --watch
npm test                   # 212 tests
npm run test:coverage      # c8 text + lcov + html
npm run lint               # eslint src tests
npm run lint:fix
npm run typecheck          # tsc --noEmit
npm run docs               # typedoc HTML
npm run docs:md            # typedoc-plugin-markdown
npm run clean              # rm -rf dist docs/api docs/md
```

## Testing

```bash
npm test                   # mocha (212 tests)
npm run test:watch
npm run test:coverage      # c8 with text / lcov / html reports and 70/74/72/70 thresholds
```

## Build

```bash
npm run build              # rollup ESM + CJS + .d.ts → dist/
npm run prepublishOnly     # build + test gate (run automatically before publish)
```

Artifacts:
- `dist/index.mjs`, `dist/index.cjs`, `dist/index.d.ts` — library
- `dist/cli.mjs` — `vrp-solver` binary

## Release

```bash
npm run lint && npm run typecheck && npm test && npm run build
# Bump version in package.json (e.g. 0.1.2 → 0.1.3)
git tag v0.1.X && git push origin v0.1.X
# .github/workflows/publish.yml publishes to npm via trusted publishing
```

## Project Structure

```
src/
├── core/                          # Problem & solution definitions
│   ├── problem.ts                 #   - VRP-RPD problem
│   ├── solution.ts                #   - Solution routing
│   ├── multi-depot-problem.ts     #   - Multi-depot
│   ├── traffic-aware-problem.ts   #   - Traffic model
│   ├── resource-transfer.ts       #   - Inter-vehicle transfers
│   ├── vehicle-with-capabilities.ts
│   └── solution-with-transfers.ts
├── algorithms/
│   ├── alns/                      # ALNS metaheuristic
│   │   ├── alns.ts
│   │   ├── operators.ts
│   │   └── transfer-aware-operators.ts
│   └── brkga/                     # BRKGA evolutionary algorithm
│       ├── brkga.ts
│       ├── decoder.ts
│       └── island-messenger.ts    #   - Worker communication
├── analytics/                     # Solution analysis
│   ├── route-analytics.ts
│   └── solution-comparator.ts
├── export/                        # GIS export (GeoJSON, KML, CSV)
│   └── gis-exporter.ts
├── errors.ts                      # Typed error classes
├── logger.ts                      # Logger interface
├── cli.ts                         # CLI entry point
├── index.ts                       # Public API exports
├── worker.ts                      # Worker thread entry point
└── worker-validation.ts           # Worker data validation

samples/                           # Example problem files
├── basic.json
├── time-windows.json
├── multi-depot.json
└── delhi-10.json
```

## Tech Stack

| Category | Technology |
|----------|------------|
| Language | TypeScript 5.7+ |
| Module system | ES Modules with CommonJS + `.d.ts` support |
| Runtime | Node.js ≥ 18 (engines; reference badge targets 20+) |
| Build | Rollup + `@rollup/plugin-typescript` + `rollup-plugin-dts` |
| Test framework | Mocha + Chai |
| Coverage | c8 |
| Lint | ESLint + `@typescript-eslint` + `eslint-plugin-import` |
| Type check | `tsc --noEmit` |
| Docs | TypeDoc + `typedoc-plugin-markdown` |
| CLI | `dist/cli.mjs` (`vrp-solver` bin) |
| Parallelism | `worker_threads` |
| Dev runner | Vite (dev mode only) |

## Roadmap

- **v0.1.x** — Current: ALNS + BRKGA two-stage solver, time windows, multi-depot, traffic-aware routing, transfer-aware operators, GIS export, island-model parallelism.
- **v0.2.0** — Planned: lower-cost transfer models, dynamic stochastic customer arrivals, mid-route re-optimisation triggers.
- **v0.3.0** — Planned: GPU-accelerated BRKGA fitness evaluation, REST gateway, GeoJSON streaming.
- **v1.0.0** — Planned: full parity with paper benchmarks on Indian cities; scenario replay; production sample bundles.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## Code of Conduct

This project follows the [Contributor Covenant v2.1](CODE_OF_CONDUCT.md).

## Security

Report vulnerabilities to **sachncs@gmail.com** — see [SECURITY.md](SECURITY.md).

## License

[ISC](LICENSE) © 2026 Sachin

## References

- Saseendran, H., Sodhi, M., & Prasad, R. (2026). *Vehicle Routing Problem with Resource-Constrained Pickup and Delivery*. [arXiv:2602.23685](https://arxiv.org/abs/2602.23685) · [HTML](https://arxiv.org/html/2602.23685v2)
