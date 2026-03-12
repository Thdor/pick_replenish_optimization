# OEM Warehouse Optimization Model — Handover Guide

> **Purpose**: This document explains the complete business logic and mathematics behind the operational efficiency model, so that a new team member can understand, operate, and extend it without reading source code.

---

## Table of Contents

1. [Business Context](#1-business-context)
2. [Model Overview](#2-model-overview)
3. [Pipeline Architecture](#3-pipeline-architecture)
4. [Step 1 — Data Validation](#4-step-1--data-validation)
5. [Step 2 — Demand Profiling](#5-step-2--demand-profiling)
6. [Step 3 — ABC-XYZ Classification & Weighting](#6-step-3--abc-xyz-classification--weighting)
7. [Step 4 — Travel Model & Model Inputs](#7-step-4--travel-model--model-inputs)
8. [Step 5 — DOC Optimization](#8-step-5--doc-optimization)
9. [Step 7 — Day-by-Day Simulation](#9-step-7--day-by-day-simulation)
10. [Story Charts & Visual Outputs](#10-story-charts--visual-outputs)
11. [Key Parameters Reference](#11-key-parameters-reference)
12. [How to Run](#12-how-to-run)
13. [Input Data Requirements](#13-input-data-requirements)
14. [Output Files](#14-output-files)
15. [Known Limitations & Assumptions](#15-known-limitations--assumptions)

---

## 1. Business Context

### The Problem

In a warehouse, each SKU can be stored in two locations:

- **Pick face (forward pick zone)**: Ergonomic, close to packing stations — fast to pick from, but limited space.
- **Reserve (bulk storage)**: Farther away — slower and more expensive travel per pick.

When a picker needs an item, they prefer the pick face. But the pick face must be **replenished** from reserve whenever stock runs low. This creates a fundamental trade-off:

| Give a SKU MORE pick-face space | Give a SKU LESS pick-face space |
|---|---|
| Fewer replenishment trips needed | More replenishment trips needed |
| But: pick zone becomes larger → pickers walk farther | But: pick zone stays compact → pickers walk less |

### The Decision Variable

For each SKU *i*, the model decides its **DOC (Days of Cover)** in the pick face — how many days' worth of average demand to keep at the pick face. A higher DOC means more inventory in the pick zone, fewer replenishments, but a bigger zone to walk through.

### The Goal

**Minimize the total daily operating cost**, which is the sum of:
- **Travel cost**: what it costs pickers to walk to each item
- **Replenishment cost**: what it costs to move inventory from reserve to pick face

This model applies to a Vietnamese e-commerce warehouse with ~20,000 SKUs and ~2.8 million order lines per month.

---

## 2. Model Overview

### The Objective Function

$$
\text{Minimize } \quad \text{TotalCost} = \underbrace{c_{\text{replen}} \times \sum_{i=1}^{N} \frac{w_i}{\text{DOC}_i}}_{\text{Replenishment Cost}} + \underbrace{c_{\text{travel}} \times \bar{T}_{\text{eff}}(F) \times \text{OL}_{\text{total}}}_{\text{Travel Cost}}
$$

Where:
| Symbol | Meaning |
|---|---|
| $c_{\text{replen}}$ | Cost per replenishment trip ($) |
| $c_{\text{travel}}$ | Cost per meter of picker travel ($) |
| $w_i$ | Priority weight for SKU *i* (from ABC-XYZ classification) |
| $\text{DOC}_i$ | Days of cover in the pick face for SKU *i* (the decision variable) |
| $\bar{T}_{\text{eff}}(F)$ | Average effective travel distance per pick, a function of total pick zone volume |
| $F$ | Total pick zone volume in m³ = $\sum_i \text{DOC}_i \times d_i \times v_i$ |
| $\text{OL}_{\text{total}}$ | Total order lines per day across all SKUs |
| $d_i$ | Average daily demand (units) for SKU *i* |
| $v_i$ | Unit volume (m³) for SKU *i* |

### How the Two Costs Interact

- **Replenishment cost** decreases as DOC increases (fewer trips: each SKU is replenished every DOC days → trips/day = $w_i$ / DOC$_i$).
- **Travel cost** increases as DOC increases (more inventory in pick zone → bigger zone → longer walks).

The optimizer finds the DOC per SKU that balances these forces, subject to space and replenishment capacity constraints.

---

## 3. Pipeline Architecture

The model runs as a sequential pipeline of 6 steps (step 6 is not used):

```
Step 1 → Step 2 → Step 3 → Step 4 → Step 5 → Step 7
 │          │         │         │         │         │
Validate  Profile   Classify  Travel    Optimize  Simulate
 Data     Demand    ABC-XYZ   Model    DOC per    Day-by-
                    + Weight   + Prep    SKU       Day
```

After the pipeline, 30 story charts are generated across 8 chapters.

---

## 4. Step 1 — Data Validation

### What it does

Loads and validates all 5 input CSV files, cleaning invalid records:

| File | Purpose |
|---|---|
| `sku_master.csv` | Physical dimensions, weight, case pack per SKU |
| `order_lines.csv` | Historical order line transactions |
| `warehouse_layout.csv` | Slot types, counts, and dimensions per zone |
| `warehouse_geometry.csv` | Physical measurements of the warehouse (aisle length, width, etc.) |
| `parameters.csv` | Operational parameters (wages, speeds, etc.) |

### Key cleaning rules
- Drops SKUs with invalid `case_pack_qty` (zero or null)
- Drops order lines for "orphan" SKUs not in the master
- Computes `slot_volume_cm3` for each slot type

---

## 5. Step 2 — Demand Profiling

### What it does

Builds a daily demand matrix (SKU × Date) and computes per-SKU statistics that drive the entire model.

### Per-SKU metrics computed

| Metric | Formula | Purpose |
|---|---|---|
| $d_i$ (avg daily demand) | total_units / total_days | Core input to DOC formula |
| $\text{OL}_i$ (avg daily order lines) | total_order_lines / total_days | Drives travel cost |
| $\sigma_i$ (std daily demand) | Standard deviation of daily units | Used for CV |
| $\text{CV}_i$ (coefficient of variation) | $\sigma_i / d_i$ | Measures demand variability → XYZ class |
| active_days | Count of days with demand > 0 | Data quality filter |

### SKU filtering (flags)

SKUs are categorized into three groups:

| Flag | Condition | Treatment |
|---|---|---|
| **ok** (active) | Has positive demand AND ≥ 5 active days | Included in optimization |
| **insufficient_data** | Has positive demand BUT < 5 active days | Excluded — not enough data for reliable DOC |
| **dead_stock** | Zero total demand | Excluded — no demand to optimize |

**Impact**: In the real dataset (January 2026, 31 days), out of 20,821 SKUs:
- 5,247 active → cover **98.3%** of all order lines
- 15,440 insufficient data → only **1.7%** of order lines (11,900+ of them had demand on just 1 day)
- 134 dead stock → 0% of order lines

The filtering is aggressive but appropriate: the excluded SKUs are long-tail items that would add noise to the optimization with negligible impact.

### Key output
- `01_demand_profile.csv` — per-SKU statistics with flag

### Related charts
- `ch2_01_daily_order_volume.png` — Daily order line volume over time (not units, not orders)
- `ch2_02_velocity_pareto.png` — Pareto curve showing what % of SKUs drive what % of order lines
- `ch2_03_demand_distribution.png` — Histogram of average daily demand (active SKUs only)
- `ch2_04_demand_vs_cv.png` — Scatter of demand volume vs variability, colored by ABC class

---

## 6. Step 3 — ABC-XYZ Classification & Weighting

### ABC Classification (pick velocity)

SKUs are ranked by total order lines (descending) and split into three classes:

| Class | Rule | Interpretation |
|---|---|---|
| **A** | Top 80% of cumulative order lines | High-velocity: few SKUs, many picks |
| **B** | Next 15% (80–95% cumulative) | Medium-velocity |
| **C** | Remaining 5% | Low-velocity: many SKUs, few picks |

*These thresholds (80/95) are configurable in `config.py`.*

### XYZ Classification (demand variability)

SKUs are classified by their coefficient of variation (CV = σ/μ):

| Class | Rule (percentile method) | Interpretation |
|---|---|---|
| **X** | CV below 25th percentile | Stable, predictable demand |
| **Y** | CV between 25th and 75th percentile | Moderate variability |
| **Z** | CV above 75th percentile | Highly variable, unpredictable |

### Combined ABC-XYZ matrix

Creates a 3×3 matrix (AX, AY, AZ, BX, BY, BZ, CX, CY, CZ) with informational priority labels:

| Priority | Segments | Meaning |
|---|---|---|
| **High** | AX, AY, BX | Fast AND predictable → best for pick face |
| **Medium** | AZ, BY, CX | Mixed characteristics |
| **Low** | BZ, CY, CZ | Slow or unpredictable → pick face less valuable |

**Important**: All active SKUs are assigned to the pick face regardless of priority. The priority only affects the **weight** $w_i$ in the optimization, which adjusts how aggressively each SKU's DOC is set.

### Weight computation ($w_i$)

The weight combines ABC and XYZ factors using a **hybrid log-dampening method**:

$$w_i = w_{\text{abc}} \times w_{\text{xyz}}$$

**ABC weights** (based on mean order lines per class):
$$w_A = 1 + \log_{10}\!\left(\frac{\overline{\text{OL}}_A}{\overline{\text{OL}}_B}\right) \times \text{dampening}$$
$$w_B = 1.0 \quad \text{(baseline)}$$
$$w_C = 1 - \log_{10}\!\left(\frac{\overline{\text{OL}}_B}{\overline{\text{OL}}_C}\right) \times \text{dampening}$$

**XYZ weights** (based on median CV per class):
$$w_X = 1 - \log_{10}\!\left(\frac{\widetilde{\text{CV}}_Y}{\widetilde{\text{CV}}_X}\right) \times \text{dampening}$$
$$w_Y = 1.0 \quad \text{(baseline)}$$
$$w_Z = 1 + \log_{10}\!\left(\frac{\widetilde{\text{CV}}_Z}{\widetilde{\text{CV}}_Y}\right) \times \text{dampening}$$

The **dampening** factor (default 0.3) controls how much the ABC-XYZ class affects the weight. A value of 0 means no class effect (all weights = 1); higher values amplify the difference.

**What the weight does**: A higher weight makes the optimizer give that SKU a **lower DOC** (more frequent replenishments), because the cost of running out of stock at the pick face is weighted higher. In practice:
- A-class SKUs get $w > 1$ → shorter DOC → more replenishments but always available at pick face
- C-class SKUs get $w < 1$ → longer DOC → fewer replenishments but larger pick face allocation

**Example weight matrix** (from real data):

|  | X | Y | Z |
|--|---|---|---|
| **A** | 1.155 | 1.259 | 1.356 |
| **B** | 0.918 | 1.000 | 1.077 |
| **C** | 0.695 | 0.758 | 0.816 |

### Key output
- `02_abc_xyz.csv` — classified SKUs with weights
- `02_abc_xyz_summary.csv` — 3×3 summary matrix

### Related charts
- `ch3_01_abc_xyz_heatmap.png` — 3×3 matrix showing SKU count, OL share, and weight per cell
- `ch3_02_pick_candidate_split.png` — Status breakdown (active/insufficient/dead) + ABC split
- `ch3_03_order_line_share_abc.png` — Bar chart of order line share by ABC class
- `ch3_04_demand_by_abc.png` — Box plot of demand distribution per ABC class

---

## 7. Step 4 — Travel Model & Model Inputs

### The Dynamic-Aisle Travel Model

This is the core physical model that translates pick zone volume into walking distance. It answers: *"If the pick zone occupies F cubic meters, how far does a picker walk per pick?"*

#### Chain of logic

```
F (total volume) → n_aisles → route_distance → T̄_geometric → congestion → T̄_effective
```

#### Step-by-step derivation

**1. Volume → Number of aisles**

$$n_{\text{aisles}} = \frac{F}{V_{\text{aisle}}}$$

Where $V_{\text{aisle}} = \text{aisle\_length} \times \text{rack\_depth} \times \text{usable\_height} \times 2$ (both sides of the aisle). This is fractional — a pick zone doesn't fill whole aisles.

**2. Aisles → Expected route distance (birthday problem)**

For a picking route of $P$ picks distributed across $n$ aisles, the expected number of aisles visited is:

$$E_{\text{aisles}} = n \times \left(1 - \left(\frac{n-1}{n}\right)^P\right)$$

This is the birthday-problem formula: what's the expected number of distinct aisles hit when randomly distributing $P$ picks.

Route distance then:

$$\text{route\_distance} = E_{\text{aisles}} \times L_{\text{aisle}} + (E_{\text{aisles}} - 1) \times W_{\text{aisle}} + 2 \times W_{\text{cross}}$$

Where $L_{\text{aisle}}$ is aisle length, $W_{\text{aisle}}$ is aisle width (cross-travel between aisles), and $W_{\text{cross}}$ is the cross-aisle distance (detour to enter/exit the pick block).

**3. Route distance → Geometric travel per pick**

$$\bar{T}_{\text{geo}} = \beta + \frac{\text{route\_distance}}{P}$$

Where $\beta = \frac{2 \times \text{staging\_distance}}{P}$ captures the round-trip from the staging area to the pick block.

**4. Geometric → Effective (with congestion)**

$$\bar{T}_{\text{eff}} = \bar{T}_{\text{geo}} \times \left(1 + \left(\frac{\text{aisle\_traffic}}{K}\right)^2\right)$$

Where:
- $\text{aisle\_traffic} = \frac{\text{OL}_{\text{total}} / \text{picking\_hours}}{n_{\text{aisles}}}$ — picks per aisle per hour
- $K = \frac{\text{max\_comfortable} \times 3600}{\text{time\_in\_aisle}}$ — aisle capacity in picks/hour
- $\text{max\_comfortable} = \frac{\text{aisle\_area}}{\text{picker\_comfortable\_space}}$ — how many pickers fit comfortably in one aisle

The congestion term is a **quadratic penalty**: when aisles are lightly loaded, it's ≈1 (no effect). As aisle traffic approaches capacity K, congestion grows rapidly.

**Why this matters**: If you shrink the pick zone (fewer aisles), geometric distance decreases, but aisle congestion increases because the same traffic is squeezed into fewer aisles. The model captures this trade-off.

### Marginal cost of space

The optimizer needs to know: *"What's the cost of adding 1 more m³ to the pick zone?"*

$$c_{\text{marginal}} = c_{\text{travel}} \times \text{OL}_{\text{total}} \times \frac{d\bar{T}_{\text{eff}}}{dF}$$

This is computed numerically (finite difference) and fed into the DOC formula.

### Space burn rate

Each SKU's **space burn rate** = $d_i \times v_i$ (m³/day per DOC day). This tells us how much pick-zone space is "consumed" per day of cover:

- SKU with $d_i = 100$ units/day and $v_i = 0.001$ m³/unit → burns 0.1 m³ per DOC day
- That same SKU at DOC = 5 → occupies 0.5 m³ in the pick face

### Key output
- `03_model_inputs.csv` — per-SKU inputs for optimizer (with $d_i$, $v_i$, $w_i$, burn rate)
- `03_model_params.csv` — global parameters (travel constants, costs, capacities)

### Related charts
- `ch4_01_travel_curve.png` — T̄(F) curve showing how travel distance changes with pick zone volume, with the optimal F marked and a congestion multiplier on the right axis
- `ch4_02_space_burn_rate.png` — Distribution of space burn rates by ABC class (log scale, with box plot)

---

## 8. Step 5 — DOC Optimization

### Per-SKU closed-form DOC

For each SKU, the optimal DOC has a closed-form solution (derived from the first-order optimality condition):

$$\text{DOC}_i = \sqrt{\frac{c_{\text{replen}} \times w_i}{c_{\text{marginal}} \times d_i \times v_i}}$$

**Clipped** to [1.0, 15.0] days — no SKU gets less than 1 day or more than 15 days of cover.

**Intuition**:
- **High $w_i$ (important SKU)** → higher DOC → more stock at pick face → always available
- **High $d_i \times v_i$ (space-hungry SKU)** → lower DOC → less space consumed
- **High $c_{\text{marginal}}$ (adding space is expensive)** → lower DOC → conserve space

### Solver structure

The DOC formula depends on $c_{\text{marginal}}$, which itself depends on $F$ (total pick zone volume), which depends on all the DOC values. This circular dependency is resolved by iteration:

```
OUTER LOOP (converges F, typically 4-5 iterations):
  │
  ├─ Compute c_marginal at current F
  │
  ├─ INNER LOOP (adjusts Lagrangian multipliers):
  │    │
  │    ├─ If replen trips > MAX_REPLEN_TRIPS_PER_DAY:
  │    │     increase λ_replen → pushes DOC up → fewer trips
  │    │
  │    ├─ If space mode = "fixed" AND F > F_capacity:
  │    │     increase λ_space → pushes DOC down → less space
  │    │
  │    └─ Compute DOC per SKU (closed-form)
  │
  └─ Recompute F from new DOC values → check convergence (|F_new - F| / F < 0.5%)
```

### Constraints

| Constraint | Mechanism |
|---|---|
| **Space**: $\sum_i \text{DOC}_i \times d_i \times v_i \le F_{\text{cap}}$ | In "fixed" mode: Lagrangian multiplier $\lambda_{\text{space}}$ penalizes excess space. In "optimize" mode (default): no hard cap, model finds natural equilibrium. |
| **Replenishment**: $\sum_i 1/\text{DOC}_i \le \text{MAX\_R}$ | Lagrangian multiplier $\lambda_{\text{replen}}$ pushes DOC up when too many replen trips. |
| **DOC bounds**: $1.0 \le \text{DOC}_i \le 15.0$ | Hard clip after each computation. |

### Benchmark comparison

After optimization, the model evaluates 4 alternative "naive" strategies for comparison:

| Strategy | DOC rule | Purpose |
|---|---|---|
| **Uniform** | All SKUs at mean of optimized DOC | What if everyone got the same DOC? |
| **Max (15d)** | All at maximum | What if we maximized pick-face stock? |
| **Min (1d)** | All at minimum | What if we minimized pick-face stock? |
| **ABC heuristic** | A=3d, B=7d, C=15d | Common industry rule-of-thumb |

### Key output
- `04_optimal_allocation.csv` — per-SKU DOC, pick face volume, replen frequency
- `05_benchmarks.csv` — cost comparison across strategies
- `07_convergence_log.csv` — F, costs, and parameters at each iteration

### Related charts
- `ch5_01_cost_tradeoff.png` — Replen vs travel vs total cost across iterations, with optimal point marked
- `ch5_02_convergence_F.png` — Pick zone volume converging across iterations
- `ch5_03_convergence_cost.png` — Total cost converging
- `ch5_04_doc_by_abc.png` — DOC distribution by ABC class (A gets lower DOC, C gets higher)
- `ch5_05_pick_face_volume_abc.png` — Space allocation by ABC class
- `ch5_06_replen_freq_abc.png` — Replenishment frequency by ABC class
- `ch5_07_constraint_utilization.png` — Space and replen utilization vs capacity
- `ch6_01_benchmark_total_cost.png` — Total cost comparison against naive strategies
- `ch6_02_benchmark_breakdown.png` — Cost breakdown (travel vs replen) by strategy
- `ch6_03_benchmark_replen.png` — Replen trips by strategy vs capacity limit
- `ch6_04_benchmark_space.png` — Space usage by strategy vs capacity

---

## 9. Step 7 — Day-by-Day Simulation

### Why simulate?

The optimizer uses average daily values. The simulation **replays actual historical demand day-by-day** to validate that the optimized DOC allocation works under real-world variability: demand spikes, quiet days, and uneven SKU activity.

### Baseline scenario (no pick face)

In the baseline, no pick-face allocation exists. Every order line requires a trip to the **reserve zone**:

$$\text{Baseline daily cost} = c_{\text{travel}} \times \text{reserve\_travel} \times \text{order\_lines}_{\text{day}}$$

Where:
$$\text{reserve\_travel} = 2 \times (\text{aisle\_length} + \text{staging\_distance})$$

This is a round trip: staging → reserve location → back to staging. All picks are equally expensive.

### Proposed scenario (with optimized pick face)

Each day, the simulation processes each SKU:

1. **Check pick face**: If pick face has enough stock for the day's demand → **pick zone hit** (short travel at $\bar{T}_{\text{eff}}$)
2. **Partial stock**: If some stock remains but not enough → split order lines proportionally between pick zone and reserve
3. **Empty pick face**: All order lines served from reserve (long travel)
4. **Replenishment trigger**: If pick face drops below ~1 hour of stock ($\max(1, d_i / \text{picking\_hours})$), the SKU becomes a replen candidate
5. **Replenishment prioritization**: Candidates sorted by $w_i \times \text{urgency}$, where urgency = how far below the trigger. Only MAX_R trips per day allowed.

$$\text{Proposed daily cost} = c_{\text{travel}} \times \bar{T}_{\text{eff}} \times \text{pick\_hits} + c_{\text{travel}} \times \text{reserve\_travel} \times \text{reserve\_picks} + c_{\text{replen}} \times \text{replen\_trips}$$

### Key KPIs

| KPI | Meaning |
|---|---|
| **Pick zone hit rate** | % of order lines served from pick face (target: >70%) |
| **Savings %** | (Baseline − Proposed) / Baseline × 100 |
| **Avg replen trips/day** | Should stay within MAX_R capacity |
| **Stockout events** | Order lines that couldn't be served from pick face when it was empty (informational) |

### Key output
- `06_simulation_kpis.csv` — summary KPIs
- `06_baseline_daily.csv` — daily baseline metrics
- `06_proposed_daily.csv` — daily proposed metrics

### Related charts
- `ch7_01_daily_cost_comparison.png` — Baseline vs proposed cost time series with shaded savings
- `ch7_02_cumulative_savings.png` — Running total of savings over time
- `ch7_03_daily_replen.png` — Daily replenishment trips vs capacity
- `ch7_04_pick_zone_hit_rate.png` — Hit rate over time
- `ch7_05_stockouts.png` — Daily stockout events

---

## 10. Story Charts & Visual Outputs

All charts are generated in `output/story/`. Here is the complete catalog organized by chapter:

### Chapter 1 — The Warehouse Challenge (1 chart)

| Chart | File | Shows |
|---|---|---|
| Warehouse At-a-Glance | `ch1_01_overview.png` | Hero metrics: active SKUs, removed/inactive count, avg daily OL, ABC-A count, savings % |

### Chapter 2 — Demand Landscape (4 charts)

| Chart | File | Shows |
|---|---|---|
| Daily Order Lines Over Time | `ch2_01_daily_order_volume.png` | Time series of daily order line count (not units, not orders) with mean line |
| SKU Velocity Pareto | `ch2_02_velocity_pareto.png` | Cumulative % of order lines vs % of SKUs. The actual data point where 80% of OL is reached is highlighted (not a generic 80/20 line) |
| Avg Daily Demand Distribution | `ch2_03_demand_distribution.png` | Histogram of dᵢ for active SKUs only, clipped at P95 |
| Demand Volume vs Variability | `ch2_04_demand_vs_cv.png` | Scatter of dᵢ vs CVᵢ, colored by ABC class (log-scale x-axis) |

### Chapter 3 — SKU Segmentation (4 charts)

| Chart | File | Shows |
|---|---|---|
| ABC-XYZ Heatmap | `ch3_01_abc_xyz_heatmap.png` | 3×3 matrix with SKU count, OL share %, and weight per cell |
| Pick Candidate Split | `ch3_02_pick_candidate_split.png` | Two donuts: (1) status breakdown (active/insufficient/dead), (2) active by ABC with OL share |
| Order Line Share by ABC | `ch3_03_order_line_share_abc.png` | Bar chart of total order lines per ABC class |
| Demand Box Plot by ABC | `ch3_04_demand_by_abc.png` | Box plot of avg daily demand per class |

### Chapter 4 — Travel Model & Inputs (2 charts)

| Chart | File | Shows |
|---|---|---|
| Dynamic-Aisle Travel Curve | `ch4_01_travel_curve.png` | T̄_eff and T̄_geo vs F, zoomed to relevant range around the optimized F, with congestion multiplier on right axis |
| Space Burn Rate | `ch4_02_space_burn_rate.png` | Two panels: (1) log-scale histogram of burn rates by ABC, (2) log-scale box plot by ABC |

### Chapter 5 — Optimization Results (7 charts)

| Chart | File | Shows |
|---|---|---|
| Cost Tradeoff | `ch5_01_cost_tradeoff.png` | Replen, travel, and total cost vs F across iterations — optimal point marked with star |
| Convergence: Volume | `ch5_02_convergence_F.png` | F (m³) across outer iterations |
| Convergence: Cost | `ch5_03_convergence_cost.png` | Total cost across iterations |
| DOC by ABC Class | `ch5_04_doc_by_abc.png` | Box plot of optimized DOC per class with min/max bounds |
| Pick Face Volume by ABC | `ch5_05_pick_face_volume_abc.png` | Space allocated per class |
| Replen Frequency by ABC | `ch5_06_replen_freq_abc.png` | Box plot of trips/day per class |
| Constraint Utilization | `ch5_07_constraint_utilization.png` | Space & replen usage vs capacity |

### Chapter 6 — Benchmark Comparison (4 charts)

| Chart | File | Shows |
|---|---|---|
| Total Cost Comparison | `ch6_01_benchmark_total_cost.png` | Bar chart: optimized vs 4 naive strategies |
| Cost Breakdown | `ch6_02_benchmark_breakdown.png` | Stacked bar: travel + replen per strategy |
| Replenishment Load | `ch6_03_benchmark_replen.png` | Replen trips per strategy vs MAX capacity |
| Space Usage | `ch6_04_benchmark_space.png` | Space per strategy vs capacity |

### Chapter 7 — Simulation Results (5 charts)

| Chart | File | Shows |
|---|---|---|
| Daily Cost: Baseline vs Proposed | `ch7_01_daily_cost_comparison.png` | Two time series with shaded savings area |
| Cumulative Savings | `ch7_02_cumulative_savings.png` | Running total of savings over the period |
| Daily Replenishment Trips | `ch7_03_daily_replen.png` | Bar chart with mean and capacity line |
| Pick Zone Hit Rate | `ch7_04_pick_zone_hit_rate.png` | Hit rate % over time with mean |
| Stockout Events | `ch7_05_stockouts.png` | Daily stockout count |

### Chapter 8 — Executive Summary (3 charts)

| Chart | File | Shows |
|---|---|---|
| KPI Dashboard | `ch8_01_kpi_dashboard.png` | 8-card dashboard of key metrics |
| Cost Breakdown Pie | `ch8_02_cost_breakdown_pie.png` | Travel vs replenishment (proposed) |
| Annual Projection | `ch8_03_annual_projection.png` | Monthly, quarterly, and annual savings projection |

---

## 11. Key Parameters Reference

### Cost parameters (from `parameters.csv` or `config.py`)

| Parameter | Value (Real) | Meaning |
|---|---|---|
| PICKER_WAGE_PER_HOUR | $2.00 | Picker hourly wage |
| REPLENISHER_WAGE_PER_HOUR | $4.00 | Replenisher hourly wage |
| AVG_REPLENISHMENT_TIME_MIN | 15.0 min | Time per replen trip (includes travel + put-away) |
| PICKER_SPEED_SEC_PER_METER | 1.5 sec/m | Average walking speed |
| NUM_REPLENISHERS | 5 | Number of replenishment workers |
| REPLEN_HOURS_PER_DAY | 16.0 hrs | Daily replen shift hours |
| REPLEN_UTILIZATION_FACTOR | 0.85 | % of time actually doing replen |

### Derived cost constants

| Constant | Formula | Value |
|---|---|---|
| $c_{\text{replen}}$ | (15/60) × $4.00 | **$1.00 per trip** |
| $c_{\text{travel}}$ | (1.5/3600) × $2.00 | **$0.000833 per meter** |
| MAX_REPLEN_TRIPS_PER_DAY | 5 × 16 × 60 / 15 × 0.85 | **272 trips/day** |

### Warehouse geometry (from `warehouse_geometry.csv`)

| Parameter | Real value |
|---|---|
| aisle_length_m | 30.0 |
| aisle_width_m | 3.0 |
| cross_aisle_width_m | 3.0 |
| rack_depth_m | 0.8 |
| usable_pick_height_m | 2.0 |
| staging_distance_m | 8.0 |
| picks_per_route | 50 |
| picking_hours_per_day | 16 |

### Derived travel constants

| Constant | Value | Meaning |
|---|---|---|
| volume_per_aisle | 96.0 m³ | Pick volume per aisle (both sides) |
| K | 2,400 picks/aisle/hr | Aisle capacity before congestion |
| β | 0.32 m/pick | Staging overhead per pick |
| max_comfortable | 15 pickers/aisle | Comfortable aisle occupancy |

### Optimization parameters

| Parameter | Value | Meaning |
|---|---|---|
| DOC_MIN_PICK | 1.0 day | Minimum DOC per SKU |
| DOC_MAX_PICK | 15.0 days | Maximum DOC per SKU |
| ABC_A_THRESHOLD | 0.80 | Top 80% of OL = class A |
| ABC_B_THRESHOLD | 0.95 | Next 15% = class B |
| XYZ_PERCENTILE_X | 25 | Bottom 25% CV = class X |
| XYZ_PERCENTILE_Z | 75 | Top 25% CV = class Z |
| ABCXYZ_WEIGHT_SPREAD | 0.3 | Weight dampening factor |
| SPACE_CONSTRAINT_MODE | "optimize" | No hard space cap in default mode |

---

## 12. How to Run

### Prerequisites
- Python 3.10+
- Libraries: `pandas`, `numpy`, `matplotlib`, `seaborn`, `scipy`

### Running on real data
```
python src/run_all.py --real
```
This reads from `data/raw/`, outputs to `output/`, and generates charts in `output/story/`.

### Running on sample data (for testing)
```
python src/run_all.py
```
Generates synthetic data in `data/sample/` first, then runs the full pipeline.

### Running individual steps
Each step can be run standalone for debugging:
```
python src/step_1_validate.py
python src/step_2_profile_demand.py
python src/step_3_classify_abc_xyz.py
python src/step_4_prepare_model_inputs.py
python src/step_5_optimize.py
python src/step_7_simulate.py
```

---

## 13. Input Data Requirements

### `sku_master.csv`

| Column | Type | Description |
|---|---|---|
| sku_id | string | Unique SKU identifier |
| unit_length_cm | float | Product length in cm |
| unit_width_cm | float | Product width in cm |
| unit_height_cm | float | Product height in cm |
| unit_weight_kg | float | Product weight in kg |
| case_pack_qty | int | Units per case |

### `order_lines.csv`

| Column | Type | Description |
|---|---|---|
| order_id | string | Order identifier |
| order_date | date | Date of order |
| sku_id | string | SKU ordered |
| quantity | int | Units ordered |

### `warehouse_layout.csv`

| Column | Type | Description |
|---|---|---|
| zone | string | "Pick" or "Reserve" |
| slot_type | string | e.g., "small_bin", "large_bin" |
| total_slots | int | Number of slots of this type |
| slot_width_cm | float | Slot width |
| slot_depth_cm | float | Slot depth |
| slot_height_cm | float | Slot height |

### `warehouse_geometry.csv`

| Column | Type | Description |
|---|---|---|
| parameter | string | Parameter name (see geometry table above) |
| value | float | Parameter value |

### `parameters.csv`

| Column | Type | Description |
|---|---|---|
| parameter | string | Parameter name (wages, speeds, etc.) |
| value | float | Parameter value |

---

## 14. Output Files

All outputs are saved to the `output/` directory:

| File | Content |
|---|---|
| `01_demand_profile.csv` | Per-SKU demand statistics with activity flag |
| `02_abc_xyz.csv` | Full classified dataset with ABC, XYZ, weights, priority |
| `02_abc_xyz_summary.csv` | 3×3 ABC-XYZ summary matrix |
| `03_model_inputs.csv` | Per-SKU model inputs (volumes, burn rates, DOC bounds) |
| `03_model_params.csv` | Global travel model and cost parameters |
| `04_optimal_allocation.csv` | Per-SKU optimized DOC and pick-face volumes |
| `05_benchmarks.csv` | Cost comparison across strategies |
| `06_simulation_kpis.csv` | Summary KPIs from simulation |
| `06_baseline_daily.csv` | Day-by-day baseline cost metrics |
| `06_proposed_daily.csv` | Day-by-day proposed cost metrics |
| `07_convergence_log.csv` | Solver convergence history |
| `story/*.png` | 30 story charts (see Chapter 10) |

---

## 15. Known Limitations & Assumptions

### Assumptions

1. **Uniform pick probability**: The travel model assumes any pick is equally likely to go to any aisle. In reality, A-class items could be slotted closer to staging.
2. **Fixed travel distance**: Travel per pick depends only on total pick zone volume, not on SKU location within the zone.
3. **No batching**: Each order line is treated as an independent trip. In reality, pickers batch multiple picks per route (the model uses `picks_per_route = 50` to approximate this).
4. **No time-of-day effects**: The model uses daily aggregates, not intra-day demand patterns.
5. **No seasonal adjustment**: DOC is optimized on historical averages. Seasonal shifts would require re-running the model.
6. **Splittable case packs**: The model assumes replenishments can be in any quantity (cases can be broken). If not, DOC would need to snap to case-pack multiples.

### Limitations

1. **Replen capacity often exceeded**: The optimizer may suggest more replen trips than the 272/day capacity. In "optimize" mode, the Lagrangian partially mitigates this, but it's a soft constraint. Monitor `ch5_07_constraint_utilization.png`.
2. **Single-zone model**: The model treats the entire pick zone as one region. It doesn't model multi-zone layouts (e.g., mezzanine vs ground floor).
3. **No slotting optimization**: The model decides HOW MUCH to put in the pick face, not WHERE to place it within the pick zone.
4. **Static DOC**: The DOC is fixed for the simulation period. In practice, DOC could be dynamically adjusted based on recent demand.
5. **SKU filtering at 5 active days**: SKUs with fewer than 5 active days in the analysis period are excluded. For very short analysis windows, this may remove SKUs that should be included.

### Recommendations for the new owner

- **Re-run monthly** with fresh order data to adapt to demand changes.
- **Adjust `config.py` parameters** if operational conditions change (wages, replen time, staffing).
- **Start with "optimize" space mode** (default). Switch to "fixed" only if pick zone has a hard physical limit that must be enforced.
- **Review the cost tradeoff chart** (`ch5_01_cost_tradeoff.png`) to verify the optimizer found a clear minimum — if the curve is flat, the model has little sensitivity and results should be interpreted cautiously.
- **Use the simulation charts** (Chapter 7) as the primary validation — they show real performance, not theoretical.
