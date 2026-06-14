# VRP-RPD Solver

Route optimization for Indian logistics — delivery fleets with resource-constrained pickup and delivery.

[![License: ISC](https://img.shields.io/badge/License-ISC-yellow)](LICENSE)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.7-blue)](https://www.typescriptlang.org/)
[![Node](https://img.shields.io/badge/node-%3E%3D18-brightgreen)](https://nodejs.org/)
[![CI](https://github.com/sachn-cs/vehicle-routing-problem-with-resource-constraints/actions/workflows/ci.yml/badge.svg)](https://github.com/sachn-cs/vehicle-routing-problem-with-resource-constraints/actions/workflows/ci.yml)
[![Tests](https://img.shields.io/badge/tests-212-passing-green)]()

## Overview

This library solves the **Vehicle Routing Problem with Resource-Constrained Pickup and Delivery (VRP-RPD)** — where goods must be delivered, processed, and then picked up, potentially by different vehicles across multiple trips. It uses a two-stage metaheuristic (ALNS + BRKGA) to find high-quality routes fast.

**Built for Indian logistics:** Supports time windows, multi-depot operations, traffic-aware routing, inter-vehicle transfers at hub nodes, multi-objective optimization (cost, distance, CO₂), and island-model parallel BRKGA.

### Algorithmic Improvements Beyond the Paper

This implementation surpasses the baseline algorithms described in arXiv:2602.23685v2 with several novel enhancements:

| Improvement | Description |
|-------------|-------------|
| **Adaptive Removal Sizing** | ALNS removal fraction auto-adjusts 10%→45% based on stagnation ratio |
| **Multi-Restart ALNS** | Up to 3 restarts with temperature reset and weight zeroing on stagnation |
| **Clone Avoidance** | ALNS only clones solution on new best, avoiding regressions |
| **Elite Diversity Preservation** | Mild mutation on elite BRKGA copies proportional to stagnation |
| **Adaptive Mutation Rate** | Up to +5% extra mutants injected when population stagnates |
| **Immigrant Injection** | 20% of population replaced with fresh random individuals before breaking stagnation |
| **Hall-of-Fame Tracking** | Best-ever solution tracked separately from population elite |
| **Decoder O(1) Capacity Checks** | Incremental `RouteLoad` tracking replaces O(n) route simulation |
| **Island-Model Parallelization** | Multi-population BRKGA with elite migration via `worker_threads` |

## Quick Start

### Install

```bash
npm install vehicle-routing
```

### Solve a Problem

```typescript
import { VrpRpdSolver, VrpProblem, LocationNode, Customer, Vehicle } from 'vehicle-routing';

// Define your delivery network
const nodes = {
  0: new LocationNode(0, 0, 0, 'Depot'),            // warehouse
  1: new LocationNode(1, 10, 0, 'Customer A - Drop'), // delivery point
  2: new LocationNode(2, 20, 0, 'Customer A - Pick'), // pickup point
};

// Each customer needs a delivery and later a pickup
const customers = [new Customer(1, 1, 2, 50)]; // id, del-node, pk-node, processing-minutes
const vehicles = [new Vehicle(1, 5)];           // id, capacity

const problem = new VrpProblem(nodes, customers, vehicles, 0);
const solver = new VrpRpdSolver(problem);

const solution = await solver.solve({
  maxTimeMs: 30000,    // stop after 30 seconds
});

console.log(`Best makespan: ${solution.makespan.toFixed(2)} min`);
console.log(`Feasible: ${solution.isFeasible()}`);
console.log(`Distance: ${solution.totalDistance.toFixed(2)} km`);
```

### Via CLI

```bash
# Install globally
npm install -g vehicle-routing

# Solve a problem file
vrp-solver --problem samples/delhi-10.json --output solution.json

# Show progress
vrp-solver --problem samples/mumbai-20.json --progress
```

## Features

| Feature | Description |
|---------|-------------|
| **ALNS** | Adaptive Large Neighborhood Search — 6 destroy + 4 repair operators, adaptive weight selection |
| **BRKGA** | Biased Random-Key Genetic Algorithm — 4n chromosome, elite/mutant/crossover evolution |
| **Warm-start** | ALNS solution seeds 15% of BRKGA population for faster convergence |
| **Time Windows** | Earliest/latest delivery and pickup constraints (VRPTW) |
| **Multi-Depot** | Vehicles can start/end at different depots |
| **Traffic-Aware** | Time-dependent travel speeds via traffic model |
| **Inter-Vehicle Transfers** | Exchange resources at hub nodes |
| **Multi-Objective** | Pareto optimization for makespan, distance, cost, CO₂ |
| **Analytics** | Vehicle utilization, wait times, load profiles, route comparison |
| **GIS Export** | GeoJSON, KML, CSV output for QGIS, Google Earth, Excel |
| **Serialization** | Save/load solutions as JSON |
| **Parallel Solving** | Run ALNS and BRKGA concurrently via worker threads |
| **Progress Callback** | Real-time progress with iteration and best makespan |

## CLI

```bash
vrp-solver [options]

Options:
  --problem <file>          Path to problem JSON file (required)
  --output <file>           Write solution JSON (default: stdout)
  --alns-iterations <n>     ALNS iterations (default: 500)
  --population-size <n>     BRKGA population size (default: 30000)
  --max-generations <n>     BRKGA max generations (default: 20000)
  --max-time <ms>           Max solver time, 0 = unlimited (default: 0)
  --target-makespan <n>     Early stopping target (default: 0)
  --parallel                Run ALNS + BRKGA in parallel
  --no-warm-start           Disable ALNS warm-start
  --progress                Print progress to stderr
  --version                 Print version
  --help                    Show help
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

For time windows, add these fields to customers:
```json
{ "id": 1, "deliveryNodeId": 1, "pickupNodeId": 2, "processingTime": 30,
  "earliestDeliveryTime": 360, "latestDeliveryTime": 480,
  "earliestPickupTime": 420, "latestPickupTime": 600 }
```

## Examples

### With Time Windows

```typescript
import { CustomerWithTimeWindows } from 'vehicle-routing';

// Customer must be delivered between 9 AM and 1 PM (360-480 min)
const customer = new CustomerWithTimeWindows(
  1, 1, 2,      // id, delivery node, pickup node
  30,            // 30 min processing
  360, 480,      // earliest/latest delivery (minutes from midnight)
  420, 600,      // earliest/latest pickup
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
const geojson = exporter.toGeoJSON();  // For QGIS/Mapbox
const kml = exporter.toKML();          // For Google Earth
const csv = exporter.toCSV();          // For Excel
```

### Progress Tracking

```typescript
const solution = await solver.solve({
  onProgress: (p) => {
    console.log(`[${p.stage}] ${p.iteration}/${p.maxIterations} — best: ${p.bestMakespan.toFixed(1)}min`);
  },
});
```

## API Reference

### Solver Options

```typescript
interface SolveOptions {
  alnsIterations?: number;     // Default: 500
  populationSize?: number;      // Default: 30000
  maxGenerations?: number;      // Default: 20000
  initialTemp?: number;         // Default: 100
  coolingRate?: number;         // Default: 0.9998
  parallel?: boolean;           // Default: false
  warmStart?: boolean;          // Default: true
  maxTimeMs?: number;           // Default: 0 (unlimited)
  targetMakespan?: number;      // Default: 0 (disabled)
  islands?: number;             // Default: 1
  migrationInterval?: number;   // Default: 50
  migrantFraction?: number;     // Default: 0.05
  logger?: Logger;
  onProgress?: (p: SolverProgress) => void;
}
```

### Island-Model BRKGA

For multi-core machines, BRKGA can run multiple populations that exchange elite individuals:

```typescript
const solution = await solver.solve({
  islands: 4,              // 4 parallel populations
  migrationInterval: 50,   // exchange elites every 50 generations
  populationSize: 30000,
  maxGenerations: 20000,
});
```

## Architecture

The solver is organized into four layers:

| Layer | Directory | Responsibility |
|-------|-----------|----------------|
| **Core** | `src/core/` | Problem definition, solution model, schedule engine |
| **Algorithms** | `src/algorithms/` | ALNS metaheuristic, BRKGA evolutionary algorithm, chromosome decoder |
| **Analytics** | `src/analytics/` | Post-solution analysis, Pareto front computation |
| **Export** | `src/export/` | GIS serialization (GeoJSON, KML, CSV) |

The `VrpRpdSolver` orchestrator (`src/index.ts:142`) runs a **two-stage metaheuristic**: ALNS first, then BRKGA warm-started from the ALNS solution. In parallel mode, both run concurrently via `worker_threads` and the best result is returned.

```
Problem ──► ALNS (adaptive destroy/repair) ──► warm-start ──► BRKGA (evolutionary) ──► Best Solution
                                                    │
                                              (chromosome encoding
                                               of ALNS solution,
                                               15% of initial pop)
```

## Design Philosophy

- **Strategy pattern** — ALNS destroy/repair operators (`operators.ts:36-278`) are interchangeable functions selected via weighted roulette, enabling easy addition of new operators.
- **Adaptive weighting** — Operator weights update every segment via reinforcement learning (`alns.ts:255-280`): `w[i] = (1-λ)·w[i] + λ·(score[i]/usage[i])` with λ = 0.1.
- **Simulated annealing acceptance** — Worsening solutions accepted with probability `exp((cost_current - cost_new) / temperature)` where temperature decays geometrically (`alns.ts:297-301`).
- **Biased crossover (BRKGA)** — Each child inherits each gene from the elite parent with probability 0.7 (`brkga.ts:272-283`).
- **Message-passing IPC** — Worker threads communicate via a typed request-response protocol (`island-messenger.ts`), supporting `evolve`, `inject`, and `finish` commands.
- **Logger DI** — A minimal `Logger` interface allows silent library use or custom logging without coupling to a framework.
- **Error hierarchy** — `VrpError` → `ValidationError` | `InfeasibleSolutionError` | `AlgorithmConvergenceError` (`errors.ts`).

## Code Decisions

### O(1) Capacity Checks via Incremental RouteLoad

The decoder (`decoder.ts:22-50`) tracks three values per route:

```
RouteLoad { currentLoad, minDelta, maxDelta }
```

A new operation is feasible iff:

```
initialLoadNeeded ≤ capacity  AND  peakLoad ≤ capacity
```

where `initialLoadNeeded = -newMin` and `peakLoad = initialLoadNeeded + newMax`. This avoids an O(n) scan of the entire route on every insertion.

### Adaptive Removal Sizing

The removal fraction grows from 10% to 45% as stagnation increases (`alns.ts:178-184`):

```
stagnationRatio = min(1, iterationsSinceImprovement / maxStagnation)
removalFraction = 0.1 + stagnationRatio × 0.35
```

### Multi-Restart ALNS

When `iterationsSinceImprovement ≥ maxStagnation` (5% of max iterations, min 25), ALNS restarts (`alns.ts:219-237`):
- Resets temperature: `temp = initialTemp × 0.5^restartsUsed`
- Increases cooling rate: `coolingRate = baseCooling × (1 + restartsUsed × 0.02)`
- Zeroes all operator weights
- Up to 3 restarts allowed

### Two-Pass Greedy Decoder

The 4n-gene chromosome (`decoder.ts:10-20`) is decoded in two passes:

1. **Delivery pass** — All deliveries scheduled in priority order, assigned to vehicles via `σ` genes
2. **Schedule calculation** — Determines delivery visit times (needed for pickup feasibility)
3. **Pickup pass** — Pickups scheduled respecting resource-ready-time constraints

### Chromosome Structure (4n genes, arXiv:2602.23685v2)

```
Chromosome = { priorities (π), assignments (σ), dependencies (α), transfers (β) }
```

Each gene is a float in [0, 1). Total size = 4 × numCustomers.

### Warm-Start Encoding

The ALNS solution is encoded into a chromosome (`decoder.ts:217-251`) that seeds 15% of the initial BRKGA population:

```
priorities[i] = (routeIdx × 100 + position) / (numRoutes × 100)
assignments[i] = routeIdx / numVehicles
```

### Schedule Fixed-Point Iteration

Cross-route resource dependencies require iterative schedule computation (`solution.ts:86-159`). The loop visits all routes repeatedly until node times converge (max 1000 iterations), propagating resource-ready-time updates between vehicles.

### Stagnation Resistance in BRKGA

Three mechanisms (`brkga.ts:291-340`):
- **Elite diversity preservation**: mild mutation on elite copies proportional to stagnation ratio (`rate = min(0.05, stagnationRatio × 0.1)`)
- **Adaptive mutation rate**: extra mutants = `⌊stagnationRatio × populationSize × 0.05⌋`
- **Immigrant injection**: 20% of population replaced with fresh random individuals before breaking stagnation

## The Math

### Problem Formulation (VRP-RPD)

Each customer *c* has a delivery node *D_c*, a pickup node *P_c*, and a processing time *p_c*. The resource constraint:

```
arrivalTime(P_c) ≥ arrivalTime(D_c) + p_c
```

This creates temporal dependencies: one vehicle may deliver, another may pick up, but pickup cannot start until processing completes.

### Distance and Travel Time

Euclidean distance between nodes (`problem.ts:208`):

```
distance(i, j) = √((x_i - x_j)² + (y_i - y_j)²)
```

Travel time: `distance / speed` (optionally modified by traffic model).

### ALNS — Adaptive Weights

Every `segmentSize` iterations (default 50), operator weights are updated (`alns.ts:255-275`):

```
w[i] = (1 - λ) · w[i] + λ · (score[i] / usage[i])
```

Score per operator in each iteration:
- New global best: +33
- Better than current: +9
- Accepted via SA: +13
- Rejected: 0

Probability of selecting operator *i*:

```
P(i) = w[i] / Σⱼ w[j]
```

### ALNS — SA Acceptance Probability

```
P(accept worse) = exp((cost_current - cost_new) / temperature)
```

Temperature decays: `temp ← temp × coolingRate` each iteration.

### BRKGA — Biased Crossover

Each gene of a child is inherited (`brkga.ts:204-234`):

```
child[i] = elite[i]       with probability ρ_elite (0.7)
         = nonElite[i]    with probability 1 - ρ_elite
```

### BRKGA — Evolution per Generation

1. Sort population by fitness ascending
2. Copy top `e × popSize` elites (with mild mutation)
3. Replace bottom `m × popSize` with random mutants
4. Fill remainder via biased crossover between random elite + random non-elite

### Regret Insertion (ALNS Repair)

For regret-*k* insertion (`operators.ts:364-491`):

```
regret_k(c) = cost_k(c) - cost_1(c)
```

The customer with the largest regret (most "urgent") is inserted first. If fewer than *k* routes can accommodate a customer, a fallback cost is used.

### Shaw Relatedness (ALNS Destroy)

Used by the Shaw removal operator (`operators.ts:284-305`):

```
relatedness(c₁, c₂) = ||D_c₁ - D_c₂||₂ + |time(D_c₁) - time(D_c₂)|
```

Lower relatedness = more similar = removed together.

### Multi-Objective Pareto Front

A solution is Pareto-optimal if no other solution dominates it (`SolutionComparator.ts:135-185`):

```
other dominates current iff all objectives ≤ current
  and strictly better in at least one objective:
  {makespan, totalDistance, totalCost, totalCO₂}
```

### Traffic Model

Time-dependent travel speed (`traffic-aware-problem.ts:57-66`):

```
if departureTime ≥ latest-matching factor.startTime:
    travelTime = baseTravelTime × factor.multiplier
else:
    travelTime = segment.currentTravelTime
```

Congestion level determined by ratio `newTravelTime / baseTravelTime` (low < 1.2, medium < 1.5, high < 2.0, severe otherwise).

### Resource Transfers

Transfer duration (`resource-transfer.ts:66-121`):

```
transferDuration = amount × hub.transferTimePerUnit
```

Each hub enforces a concurrency limit (max simultaneous transfers). Vehicles have directional transfer permissions (`vehicle-with-capabilities.ts:53-64`).

## Performance Tips

- **Quick test:** Use `alnsIterations: 100, populationSize: 1000, maxGenerations: 500`
- **Production:** `alnsIterations: 500, populationSize: 30000, maxGenerations: 20000` (paper defaults)
- **Time-constrained:** Set `maxTimeMs` to stop early with a feasible solution
- **Multi-core:** Enable `parallel: true` (ALNS + BRKGA concurrently) or use `islands: 4` (parallel BRKGA populations with elite migration)
- **Stagnation resistance:** The ALNS multi-restart and BRKGA adaptive mutation/immigrant injection mechanisms automatically handle most convergence issues

## Development

```bash
# Install dependencies
npm install

# Build
npm run build

# Test
npm test                    # 212 tests
npm run test:coverage       # with coverage report

# Lint & Type Check
npm run lint
npm run typecheck

# Generate API docs
npm run docs
```

## Project Structure

```
src/
├── core/                    # Problem & solution definitions
│   ├── problem.ts                    # VRP-RPD problem (kebab-case)
│   ├── solution.ts                   # Solution routing
│   ├── multi-depot-problem.ts        # Multi-depot
│   ├── traffic-aware-problem.ts      # Traffic model
│   ├── resource-transfer.ts          # Inter-vehicle transfers
│   ├── vehicle-with-capabilities.ts
│   └── solution-with-transfers.ts
├── algorithms/
│   ├── alns/                        # ALNS metaheuristic
│   │   ├── alns.ts
│   │   ├── operators.ts
│   │   └── transfer-aware-operators.ts
│   └── brkga/                       # BRKGA evolutionary algorithm
│       ├── brkga.ts
│       ├── decoder.ts
│       └── island-messenger.ts       # Worker communication
├── analytics/             # Solution analysis
│   ├── route-analytics.ts
│   └── solution-comparator.ts
├── export/                # GIS export (GeoJSON, KML, CSV)
│   └── gis-exporter.ts
├── errors.ts              # Typed error classes
├── logger.ts              # Logger interface
├── cli.ts                 # CLI entry point
├── index.ts               # Public API exports
├── worker.ts              # Worker thread entry point
└── worker-validation.ts   # Worker data validation

samples/                   # Example problem files
├── basic.json
├── time-windows.json
├── multi-depot.json
└── delhi-10.json
```

## License

ISC — see [LICENSE](LICENSE).

## References

- Saseendran, H., Sodhi, M., & Prasad, R. (2026). Vehicle Routing Problem with Resource-Constrained Pickup and Delivery. *arXiv:2602.23685*.
- [Paper on arXiv](https://arxiv.org/abs/2602.23685)
- [HTML Version](https://arxiv.org/html/2602.23685v2)
