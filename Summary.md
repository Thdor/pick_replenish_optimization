#  Pick Face Inventory Slotting Model — Handover Guide

> **Purpose**: This document explains the complete business logic and mathematics behind the Pick Face Inventory Slotting Model 
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
9.  [Key Parameters Reference](#9-key-parameters-reference)


---

## 1. Business Context

### The Problem

In a warehouse, each SKU can be stored in two locations:

- **Pick face (forward pick zone)**: Ergonomic, close to packing stations — fast to pick from, but limited space.
- **Reserve (bulk storage)**: Farther away — slower and more expensive travel per pick.

When a picker needs an item, they prefer the pick face. But the pick face must be **replenished** from reserve whenever stock runs low. This creates a fundamental trade-off:

| Give a SKU MORE pick-face space                      | Give a SKU LESS pick-face space                  |
| ---------------------------------------------------- | ------------------------------------------------ |
| Fewer replenishment trips needed                     | More replenishment trips needed                  |
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
\text{Minimize } \quad \text{TotalCost} = \underbrace{c_{\text{replen}} \times \sum_{i=1}^{N} \frac{w_i}{\text{DOC}_i}}_{\text{Replenishment Cost}} + \underbrace{c_{\text{pick}} \times (\bar{T}_{\text{eff}} \cdot \text{speed}_0 + h_0) \times \text{fatigue} \times \text{OL}_{\text{total}}}_{\text{Picking Cost (walk + handling + fatigue)}}
$$

Where:
| Symbol                     | Meaning                                                                          |
| -------------------------- | -------------------------------------------------------------------------------- |
| $c_{\text{replen}}$        | Cost per replenishment trip ($)                                                  |
| $c_{\text{pick}}$          | Picker cost per second (= picker_wage / 3600)                                    |
| $w_i$                      | Priority weight for SKU *i* (from ABC-XYZ classification)                        |
| $\text{DOC}_i$             | Days of cover in the pick face for SKU *i* (the decision variable)               |
| $\bar{T}_{\text{eff}}(F)$  | Average effective travel distance per pick, a function of total pick zone volume |
| $\text{speed}_0$           | Walking speed (sec/m)                                                            |
| $h_0$                      | Handling time per pick (sec)                                                     |
| $\text{fatigue}$           | Fatigue multiplier: $1 + \gamma \times (\text{walk\_km} / \text{threshold})^2$   |
| $F$                        | Total pick zone volume in m³ = $\sum_i \text{DOC}_i \times d_i \times v_i$       |
| $\text{OL}_{\text{total}}$ | Total order lines per day across all SKUs                                        |
| $d_i$                      | Average daily demand (units) for SKU *i*                                         |
| $v_i$                      | Unit volume (m³) for SKU *i*                                                     |

### How the Two Costs Interact

- **Replenishment cost** decreases as DOC increases (fewer trips: each SKU is replenished every DOC days → trips/day = $w_i$ / DOC$_i$).
- **Picking cost** increases as DOC increases (more inventory in pick zone → bigger zone → longer walks → more fatigue).

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

| File                     | Purpose                                                            |
| ------------------------ | ------------------------------------------------------------------ |
| `sku_master.csv`         | Physical dimensions, weight, case pack per SKU                     |
| `order_lines.csv`        | Historical order line transactions                                 |
| `warehouse_layout.csv`   | Slot types, counts, and dimensions per zone                        |
| `warehouse_geometry.csv` | Physical measurements of the warehouse (aisle length, width, etc.) |
| `parameters.csv`         | Operational parameters (wages, speeds, etc.)                       |

### Key cleaning rules
- Drops SKUs with invalid `case_pack_qty` (zero or null)
- Drops order lines for "orphan" SKUs not in the master
- Computes `slot_volume_cm3` for each slot type

---

## 5. Step 2 — Demand Profiling

### What it does

Builds a daily demand matrix (SKU × Date) and computes per-SKU statistics that drive the entire model.

### Per-SKU metrics computed

| Metric                                   | Formula                           | Purpose                                 |
| ---------------------------------------- | --------------------------------- | --------------------------------------- |
| $d_i$ (avg daily demand)                 | total_units / total_days          | Core input to DOC formula               |
| $\text{OL}_i$ (avg daily order lines)    | total_order_lines / total_days    | Drives travel cost                      |
| $\sigma_i$ (std daily demand)            | Standard deviation of daily units | Used for CV                             |
| $\text{CV}_i$ (coefficient of variation) | $\sigma_i / d_i$                  | Measures demand variability → XYZ class |
| active_days                              | Count of days with demand > 0     | Data quality filter                     |

### SKU filtering (flags)

SKUs are categorized into three groups:

| Flag                  | Condition                               | Treatment                                   |
| --------------------- | --------------------------------------- | ------------------------------------------- |
| **ok** (active)       | Has positive demand AND ≥ 5 active days | Included in optimization                    |
| **insufficient_data** | Has positive demand BUT < 5 active days | Excluded — not enough data for reliable DOC |
| **dead_stock**        | Zero total demand                       | Excluded — no demand to optimize            |

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

| Class | Rule                              | Interpretation                      |
| ----- | --------------------------------- | ----------------------------------- |
| **A** | Top 80% of cumulative order lines | High-velocity: few SKUs, many picks |
| **B** | Next 15% (80–95% cumulative)      | Medium-velocity                     |
| **C** | Remaining 5%                      | Low-velocity: many SKUs, few picks  |

*These thresholds (80/95) are configurable in `config.py`.*

### XYZ Classification (demand variability)

SKUs are classified by their coefficient of variation (CV = σ/μ):

| Class | Rule (percentile method)            | Interpretation                 |
| ----- | ----------------------------------- | ------------------------------ |
| **X** | CV below 25th percentile            | Stable, predictable demand     |
| **Y** | CV between 25th and 75th percentile | Moderate variability           |
| **Z** | CV above 75th percentile            | Highly variable, unpredictable |

### Combined ABC-XYZ matrix

Creates a 3×3 matrix (AX, AY, AZ, BX, BY, BZ, CX, CY, CZ) with informational priority labels:

| Priority   | Segments   | Meaning                                         |
| ---------- | ---------- | ----------------------------------------------- |
| **High**   | AX, AY, BX | Fast AND predictable → best for pick face       |
| **Medium** | AZ, BY, CX | Mixed characteristics                           |
| **Low**    | BZ, CY, CZ | Slow or unpredictable → pick face less valuable |

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

|       | X     | Y     | Z     |
| ----- | ----- | ----- | ----- |
| **A** | 1.155 | 1.259 | 1.356 |
| **B** | 0.918 | 1.000 | 1.077 |
| **C** | 0.695 | 0.758 | 0.816 |

### Key output
- `02_abc_xyz.csv` — classified SKUs with weights
- `02_abc_xyz_summary.csv` — 3×3 summary matrix


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

### Fatigue Model

Pickers slow down over the course of a shift. The model captures this with a **quadratic fatigue multiplier** calibrated from time-study measurements:

$$\text{fatigue}(\text{walk\_km}) = 1 + \gamma \times \left(\frac{\text{walk\_km}}{\text{threshold}}\right)^2$$

Where:
- $\text{walk\_km}$ = daily walk per picker = $\bar{T}_{\text{eff}} \times \text{OL}_{\text{total}} / n_{\text{pickers}} / 1000$
- $\text{threshold}$ = fatigue reference distance (default: 8 km)
- $\gamma$ = fatigue coefficient, auto-derived from two time-study measurements:
  - `pick_time_fresh_sec` — pick time at the start of shift (e.g., 23s)
  - `pick_time_tired_sec` — pick time at end of shift (e.g., 28s)

The derivation:
$$\gamma = \frac{t_{\text{tired}} / t_{\text{fresh}} - 1}{(\text{walk\_km}_{\text{measured}} / \text{threshold})^2}$$

The **total pick time per order line** becomes:
$$\text{pick\_time} = (\bar{T}_{\text{eff}} \times \text{speed}_0 + h_0) \times \text{fatigue}$$

Where $\text{speed}_0$ is sec/m walking speed and $h_0$ is handling time per pick (scan, grab, place).

**Picking cost** = $c_{\text{pick}} \times \text{pick\_time} \times \text{OL}_{\text{total}}$, where $c_{\text{pick}} = \text{picker\_wage} / 3600$ ($/sec).

### Number of Pickers (Auto-Derived)

The number of pickers is **not a fixed input** — it is derived from demand and available hours:

$$n_{\text{pickers}} = \left\lceil \frac{\text{OL}_{\text{total}}}{\text{picks\_per\_hour} \times \text{working\_hours}} \right\rceil$$

Where $\text{picks\_per\_hour} = 3600 / \text{pick\_time\_fresh\_sec}$. This ensures the picker count scales realistically with throughput.

### Floor Area Conversion (m³ → m²)

The model reports floor area alongside volume. The conversion uses the **aisle footprint ratio** rather than simply dividing by shelf height:

$$\text{floor\_area} = F \times \frac{\text{aisle\_footprint}}{\text{volume\_per\_aisle}}$$

Where:
- $\text{aisle\_footprint} = L_{\text{aisle}} \times (W_{\text{aisle}} + 2 \times D_{\text{rack}})$ — total floor area per aisle including racks and walking space
- $\text{volume\_per\_aisle} = L_{\text{aisle}} \times D_{\text{rack}} \times H_{\text{usable}} \times 2$ — rack storage volume per aisle

This gives a **m²/m³ ratio** (e.g., 3.33 for current geometry). The old method `F / usable_height` only gave rack-face footprint and severely underestimated floor area.

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

---

## 8. Step 5 — DOC Optimization

### Per-SKU closed-form DOC

For each SKU, the optimal DOC has a closed-form solution (derived from the first-order optimality condition):

$$\text{DOC}_i = \sqrt{\frac{c_{\text{replen}} \times w_i}{c_{\text{marginal}} \times d_i \times v_i}}$$

**Clipped** to [DOC_MIN_PICK, DOC_MAX_PICK] days (default: 3–20 days) — configurable in `parameters.csv`.

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

| Constraint                                                                | Mechanism                                                                                                                                                           |
| ------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Space**: $\sum_i \text{DOC}_i \times d_i \times v_i \le F_{\text{cap}}$ | In "fixed" mode: Lagrangian multiplier $\lambda_{\text{space}}$ penalizes excess space. In "optimize" mode (default): no hard cap, model finds natural equilibrium. |
| **Replenishment**: $\sum_i 1/\text{DOC}_i \le \text{MAX\_R}$              | Lagrangian multiplier $\lambda_{\text{replen}}$ pushes DOC up when too many replen trips.                                                                           |
| **DOC bounds**: $\text{DOC\_MIN} \le \text{DOC}_i \le \text{DOC\_MAX}$     | Hard clip after each computation (default 3–20 days).                                                                                                                |

### Benchmark comparison

After optimization, the model evaluates 4 alternative "naive" strategies for comparison:

| Strategy          | DOC rule                          | Purpose                               |
| ----------------- | --------------------------------- | ------------------------------------- |
| **Uniform**       | All SKUs at mean of optimized DOC | What if everyone got the same DOC?    |
| **Max (15d)**     | All at maximum                    | What if we maximized pick-face stock? |
| **Min (1d)**      | All at minimum                    | What if we minimized pick-face stock? |
| **ABC heuristic** | A=3d, B=7d, C=15d                 | Common industry rule-of-thumb         |

### Key output
- `04_optimal_allocation.csv` — per-SKU DOC, pick face volume, replen frequency
- `05_benchmarks.csv` — cost comparison across strategies
- `07_convergence_log.csv` — F, costs, and parameters at each iteration

---


## 9. Key Parameters Reference

### Cost & labor parameters (from `parameters.csv`)

| Parameter                  | Default | Meaning                                           |
| -------------------------- | ------- | ------------------------------------------------- |
| picker_wage_per_hour       | $2.00   | Picker hourly wage                                |
| replenisher_wage_per_hour  | $4.00   | Replenisher hourly wage                           |
| avg_replenishment_time_min | 15.0    | Time per replen trip (includes travel + put-away)  |
| picker_speed_sec_per_meter | 1.5     | Average walking speed (sec/m)                     |
| num_replenishers           | 5       | Number of replenishment workers                   |
| working_hours_per_day      | 8.0     | Shared for picking and replenishment              |
| replen_utilization_factor  | 0.85    | % of time actually doing replen                   |
| picks_per_route            | 50      | Picks per picker route (batch size)               |

### Fatigue & picking parameters (from `parameters.csv`)

| Parameter              | Default | Meaning                                          |
| ---------------------- | ------- | ------------------------------------------------ |
| handling_time_sec      | 12.0    | Handling time per pick (scan, grab, place)        |
| pick_time_fresh_sec    | 23.0    | Time-study: pick time when fresh (start of shift) |
| pick_time_tired_sec    | 28.0    | Time-study: pick time when tired (end of shift)   |
| fatigue_threshold_km   | 8.0     | Fatigue reference distance (km)                   |

> **Note**: `num_pickers` is NOT an input — it is auto-derived as `ceil(OL_total / (3600/pick_time_fresh_sec × working_hours))`.

### Derived cost constants

| Constant                 | Formula                  | Example value           |
| ------------------------ | ------------------------ | ----------------------- |
| $c_{\text{replen}}$      | (15/60) × $4.00          | **$1.00 per trip**      |
| $c_{\text{travel}}$      | (1.5/3600) × $2.00       | **$0.000833 per meter** |
| $c_{\text{pick}}$        | $2.00 / 3600             | **$0.000556 per second**|
| MAX_REPLEN_TRIPS_PER_DAY | 5 × 8 × 60 / 15 × 0.85  | **136 trips/day**       |

### Warehouse geometry (from `warehouse_geometry.csv`)

6 physical measurements only — no operational parameters:

| Parameter             | Default |
| --------------------- | ------- |
| aisle_length_m        | 15.0    |
| aisle_width_m         | 3.0     |
| cross_aisle_width_m   | 3.0     |
| rack_depth_m          | 0.5     |
| usable_pick_height_m  | 1.2     |
| staging_distance_m    | 8.0     |

### Derived travel constants

| Constant         | Formula                                                   | Example value        |
| ---------------- | --------------------------------------------------------- | -------------------- |
| volume_per_aisle | aisle_length × rack_depth × usable_height × 2              | 18.0 m³              |
| aisle_footprint  | aisle_length × (aisle_width + 2 × rack_depth)             | 60.0 m²              |
| m²/m³ ratio      | aisle_footprint / volume_per_aisle                        | 3.333                |
| K                | max_comfortable × 3600 / time_in_aisle                    | 2,400 picks/aisle/hr |
| β                | 2 × staging_distance / picks_per_route                     | 0.32 m/pick          |
| max_comfortable  | (aisle_width × aisle_length) / picker_comfortable_space   | 7.5 pickers/aisle    |

### Optimization parameters

| Parameter             | Default    | Meaning                           |
| --------------------- | ---------- | --------------------------------- |
| DOC_MIN_PICK          | 3.0 days   | Minimum DOC per SKU               |
| DOC_MAX_PICK          | 20.0 days  | Maximum DOC per SKU               |
| DOC_TOTAL             | 30 days    | Total planned inventory days      |
| ABC_A_THRESHOLD       | 0.80       | Top 80% of OL = class A           |
| ABC_B_THRESHOLD       | 0.95       | Next 15% = class B                |
| XYZ_PERCENTILE_X      | 25         | Bottom 25% CV = class X           |
| XYZ_PERCENTILE_Z      | 75         | Top 25% CV = class Z              |
| ABCXYZ_WEIGHT_SPREAD  | 0.3        | Weight dampening factor           |
| SPACE_CONSTRAINT_MODE | "optimize" | No hard space cap in default mode |
| picker_comfortable_space | 6.0 m² | Min space per picker for comfort  |

---


---


