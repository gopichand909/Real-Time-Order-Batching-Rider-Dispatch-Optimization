# Real-Time Order Batching + Rider Dispatch Optimization
### Bengaluru Urban Delivery Simulation

A discrete-event simulation of a last-mile food/grocery delivery system across 30 zones of Bengaluru. The project models Poisson order arrivals during a peak evening window and benchmarks three dispatch strategies — Baseline, Greedy Batching, and Hungarian Assignment — against SLA, fleet distance, and rider utilization metrics.

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Project Structure](#project-structure)
- [Simulation Design](#simulation-design)
- [Pipeline Overview](#pipeline-overview)
- [Algorithms](#algorithms)
- [Key Results](#key-results)
- [Stress Test](#stress-test)
- [Dependencies](#dependencies)
- [How to Run](#how-to-run)
- [Parameters](#parameters)

---

## Problem Statement

Urban delivery platforms face two coupled decisions at every dispatch cycle:

1. **Batching** — which orders to group into a single rider trip
2. **Assignment** — which rider to assign each batch to

Poor decisions on either axis inflate delivery times, violate SLAs, and waste fleet kilometres. This project simulates a 3-hour peak window (7 PM – 10 PM) in Bengaluru and compares assignment algorithms on throughput, cost, and SLA adherence.

---

## Project Structure

```
Project_Final.ipynb          # Main notebook (all steps self-contained)
Step7_Final_Evaluation_Dashboard.png   # Output: 6-panel comparison dashboard
README.md                    # This file
```

---

## Simulation Design

### City Model
30 Bengaluru zones are modelled with real (lat, lon) coordinates and density weights that bias order generation toward high-demand areas (Koramangala, Indiranagar, HSR Layout, etc.).

### Order Arrivals
Orders arrive via a **Poisson process** (λ = 100 orders/hour) over 180 minutes, giving ~300 orders per simulation run. Each order carries a destination coordinate perturbed ±0.01° around its zone centroid.

### Rider Fleet
60 riders are spawned across random zones. 40% log in at simulation start; the rest log in at a uniformly random offset within the first hour. Shift durations range from 120–240 minutes. Each rider carries capacity 2 or 3.

### SLA
Delivery must complete within **15 minutes** of order placement.

---

## Pipeline Overview

```
Step 1 — Simulation Engine & Data Model
          │
          ▼
Step 2 — Baseline (Nearest Rider, No Batching)
          │
          ▼
Step 3 — Greedy Batching + Nearest-Rider Assignment
          │
          ▼
Step 4 — Hungarian Algorithm Assignment
          │
          ▼
Step 5 — Intra-Batch Route Optimization Benchmark
          │
          ▼
Step 6 — Real-Time Discrete Event Simulation (60-sec ticks)
          │
          ▼
Step 7 — Final Evaluation Dashboard
          │
          ▼
Stress Test — Greedy vs Hungarian under overload
```

---

## Algorithms

### Step 2 — Baseline
Each order is assigned independently to the nearest available rider with no batching. Serves as the lower-bound benchmark.

### Step 3 — Greedy Batching
Orders arriving within the same **3-minute window** are grouped by geographic proximity (≤ 1.5 km). Each batch is then assigned to the nearest available rider sequentially — a locally greedy, work-conserving policy.

### Step 4 — Hungarian Assignment
Same batch formation as Greedy. Assignment uses `scipy.optimize.linear_sum_assignment` to solve the minimum-cost bipartite matching between batches and available riders globally per cycle. Guarantees the optimal assignment for a given cost matrix.

### Step 5 — Intra-Batch Routing
Three algorithms are benchmarked on 5,000 random size-3 batches:

| Algorithm | Total Dist (km) | CPU Time (ms) | vs. Optimal |
|---|---|---|---|
| Nearest Neighbor | 66,359.98 | 569.6 | +3.12% |
| NN + 2-Opt | 64,642.20 | 1,129.4 | +0.45% |
| **Brute Force** | **64,352.40** | **1,110.6** | **0%** |

**Conclusion:** For batch size ≤ 3, Brute Force is computationally trivial (max 6 permutations) and is used throughout Steps 3–6.

### Step 6 — Real-Time Simulation
A tick-based discrete event simulation advances in 60-second increments. At each tick, all pending orders are batched and assigned via Hungarian matching, then dispatched with brute-force intra-batch routing. Riders' positions and availability are updated after each trip.

---

## Key Results

| Strategy | Orders | SLA (%) | Avg Delivery (min) | Total Dist (km) | Rider Util (%) |
|---|---|---|---|---|---|
| Baseline (No Batch) | 300 | — | — | — | — |
| Greedy Batching | 300 | — | — | — | 100.0 |
| Hungarian Batching | 300 | — | — | — | 100.0 |

> Exact values populate after running the notebook. The dashboard (`Step7_Final_Evaluation_Dashboard.png`) renders all six panels including SLA adherence, delivery time distribution, fleet distance, rider utilization, cost vs. runtime tradeoff, and a per-zone order volume heatmap.

**Key insight:** Hungarian yields lower total assignment cost at comparable or better runtime for normal batch loads. Greedy is preferable only under surge conditions (see Stress Test below).

---

## Stress Test

To expose regime-dependent behaviour, the simulation is re-run with a severely overloaded fleet:

| Parameter | Baseline | Stress |
|---|---|---|
| Arrival rate λ | 100 orders/hr | 220 orders/hr |
| Fleet size | 60 riders | 30 riders |
| Batch window | 3 min | 2 min |
| Load ratio | ~5 orders/rider | ~20.5 orders/rider |

### Stress Test Results (615 orders, 30 riders)

| Metric | Greedy | Hungarian |
|---|---|---|
| Rider Utilization (%) | 100.0 | 100.0 |
| Orders Dispatched (%) | **100.0** | 55.6 |
| SLA Success (%) | **51.7** | 19.6 |
| Avg Delivery (min) | **20.26** | 27.27 |
| Total Distance (km) | 2,380.0 | **2,006.8** |
| Avg Assignment Cost/cycle (km) | 20.93 | **19.32** |

**When Greedy wins:** Under high overload, Hungarian solves only a constrained matching (batches ≤ available riders) and drops the remainder. Greedy iterates sequentially and is work-conserving — it dispatches 100% of orders vs. Hungarian's 55.6%, with a +32.1 pp SLA advantage. Hungarian retains lower fleet distance, confirming its cost-matrix optimality, but at the cost of throughput.

**Recommendation:** Use a hybrid policy — Hungarian at normal load (≤ 5:1 order-to-rider ratio), switch to Greedy when the pending batch queue significantly exceeds available riders.

---

## Dependencies

```
python >= 3.9
numpy
pandas
scipy
matplotlib
seaborn
```

Install with:

```bash
pip install numpy pandas scipy matplotlib seaborn
```

---

## How to Run

1. Clone or download the repository.
2. Open `Project_Final.ipynb` in Jupyter Lab or Jupyter Notebook.
3. Run all cells sequentially (Kernel → Restart & Run All).
4. The final dashboard is saved as `Step7_Final_Evaluation_Dashboard.png` in the working directory.

> All steps are self-contained in a single notebook. No external data files are required — all orders and riders are generated programmatically.

---

## Parameters

All key parameters are defined at the top of Step 1 and can be modified to explore different scenarios:

| Parameter | Default | Description |
|---|---|---|
| `LAMBDA_PH` | 100 | Order arrival rate (orders/hour) |
| `N_RIDERS` | 60 | Fleet size |
| `AVG_SPEED` | 20 km/h | Rider average speed |
| `SERVICE_TIME` | 5 min | Time per delivery stop |
| `BATCH_RADIUS` | 1.5 km | Max geo-distance for batch grouping |
| `BATCH_WINDOW` | 3 min | Time window for batch formation |
| `SLA_LIMIT` | 15 min | Maximum allowed delivery time |
| `DURATION_MIN` | 180 min | Simulation duration |
| `SIM_START` | 2026-04-14 19:00 | Simulation start time |
