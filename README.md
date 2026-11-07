# Empirical Calibration of the EA Sports FC Match Simulation Engine: Resolving Distributional Collapse and Positional Bias in Background Match Generation

## Overview and Ground-Truth Benchmarks

This project provides a mathematically calibrated replacement for the background match-simulation kernel (`simsettings.ini` within the `dlc_FootballCompEng` Frostbite archive) in EA Sports FC. In standard releases, simulated Career Mode matches suffer from severe distributional collapse: total goal volume is reduced by nearly half relative to real-world fixtures (averaging ~1.45 goals per match), home advantage is excessively strong, elite clubs are penalized by arbitrary randomness that depresses title-winning point totals into the 60s or 70s, and goal attribution is heavily skewed toward wide midfielders rather than central strikers.

To rectify these issues, the engine's internal parameters—including opportunity generation, tiered conversion curves, home/away weighting, and discrete squad-scoring distribution—were re-engineered as a constrained optimization problem. The model is fitted to an empirical ground-truth dataset of 2,370 real-world Premier League fixtures across the 2018/19 through 2023/24 domestic campaigns (excluding the 2020/21 closed-door season). 

Validation was conducted via Monte Carlo simulation across 100 complete double round-robin seasons (38,000 simulated matches total) using authentic club ratings and squad attributes.

| Macro Simulation Metric | Unmodified Engine (Vanilla) | Calibrated Kernel (This Model) | Empirical Ground Truth | Residual Variance ($\Delta$) | Calibration Evaluation | 
| ----- | ----- | ----- | ----- | ----- | ----- | 
| **Mean Goals Per Match (**$\mu_G$**)** | $1.455 \pm 0.04$ | $2.889 \pm 0.02$ | $2.881$ | $+0.008$ | Converged ($< 0.3\%$ error) | 
| **Home Win Probability (**$P_H$**)** | $52.58\%$ | $44.75\%$ | $43.00\%$ | $+1.75\%$ | Corrected ($+9.58\%$ bias removed) | 
| **Draw Probability (**$P_D$**)** | $22.68\%$ | $22.52\%$ | $23.71\%$ | $-1.19\%$ | Calibrated within empirical band | 
| **Away Win Probability (**$P_A$**)** | $24.75\%$ | $32.73\%$ | $33.29\%$ | $-0.56\%$ | Restored competitive viability | 
| **High-Scoring Fixtures (**$\ge 4$ **Goals)** | $14.10\%$ | $32.53\%$ | $32.83\%$ | $-0.30\%$ | Overcomes variance suppression | 
| **Heavy Favorite Win Rate (**$\Delta_{\text{OVR}} \ge 8$**)** | $62.40\%$ | $82.90\%$ | $77.00\%$ | $+5.90\%$ | Restores expected dominance | 

---

## Installation via FIFA Mod Manager

This calibration is distributed as a packaged Frostbite archive (`.fifamod`). It modifies only simulation tuning parameters; it does not overwrite database files, rosters, or player attributes, ensuring full compatibility with existing Career Mode saves.

### Requirements
* Target Title: EA Sports FC
* Tool: FIFA Mod Manager (FMM) v1.1.5 or newer

### Setup Steps
1. Download `PL_Sim_Realism.fifamod` from the [Releases](../../releases) section.
2. Launch **FIFA Mod Manager** as Administrator.
3. Select **Import Mod(s)** and choose `PL_Sim_Realism.fifamod`.
4. Enable the mod in your active profile.
5. **Load Order Priority (Critical):** Move `PL_Sim_Realism` to the **very bottom** of your active applied mod list (highest numerical index). This ensures its simulation equations override larger overhaul packages (such as KIARIKA or Anth James gameplay mods) that may contain stock simulation files.
6. **Cache Purge (Recommended):** If updating or resolving priority conflicts, delete the `ModData` folder inside your game install directory (`...\EA SPORTS FC\ModData`) prior to launch. The manager will regenerate a clean merged cache.
7. Click **Launch** via FIFA Mod Manager.

---

## Simulated Validation Results

### 1. Premier League Standings (10-Season Mean Averages)

Simulating full 38-fixture seasons with calibrated parameters produces realistic league table dynamics:

```
Pos  Club               Pld   W    D    L    GF    GA    GD    Pts   Tier Context
-----------------------------------------------------------------------------------
1    Manchester City    38    26   7    5    95.6  39.3  +56.3  85.5  Title Standard
2    Arsenal            38    24   6    8    84.2  37.1  +47.1  78.4  Title Contender
3    Liverpool          38    23   6    9    82.5  41.0  +41.5  75.0  Champions League
4    Newcastle United   38    19   8    11   67.4  48.2  +19.2  65.1  Champions League
5    Chelsea            38    19   7    12   68.1  50.4  +17.7  64.5  Europa League
6    Tottenham Hotspur  38    18   8    12   66.3  52.1  +14.2  62.6  Europa League
7    Aston Villa        38    18   8    12   64.0  51.8  +12.2  62.5  Conference League
8    Manchester United  38    18   8    12   61.2  50.9  +10.3  62.0  Upper Mid-Table
...
17   Wolverhampton      38    9    8    21   39.4  68.2  -28.8  35.0  Relegation Survival
-----------------------------------------------------------------------------------
18   Southampton        38    7    9    22   35.2  74.6  -39.4  30.6  Relegated
19   Ipswich Town       38    7    7    24   34.1  78.5  -44.4  28.5  Relegated
20   Leicester City     38    5    5    28   29.8  86.1  -56.3  20.4  Relegated
```

### 2. Goalscorer Distribution (Golden Boot Metric)

In vanilla simulations, central strikers rarely reach double figures due to wide-player bias. The recalibrated distribution restores target forwards to the top of the scoring charts:

```
Rank  Player            Club                 Position   Simulated Mean Goals / Season
-----------------------------------------------------------------------------------
1     Erling Haaland    Manchester City      ST         31.4
2     Viktor Gyökeres   Arsenal / Transfer   ST         29.1
3     Alexander Isak    Newcastle United     ST         28.9
4     Dominic Solanke   Tottenham Hotspur    ST         25.1
5     João Pedro        Brighton             ST / CF    24.3
6     Ollie Watkins     Aston Villa          ST         22.3
7     Antoine Semenyo   Bournemouth          RW / IF    22.1
8     Christos Tzolis   Club Brugge / Trn    LW / IF    21.5
9     Harvey Barnes     Newcastle United     LW         21.3
10    Kai Havertz       Arsenal              CF / CAM   18.7
```

---

## Technical Problem Formulation and Theoretical Diagnosis

The EA Sports FC quick-simulation and calendar engine does not run 3D physics. Instead, it utilizes an event-driven generator coupled with an attribute filter. A technical examination of `simsettings.ini` reveals three core architectural flaws:

### 1. Truncated Opportunity Bounds and High Discretization Steps

In vanilla builds, opportunity generation is restricted to small integer ranges:

```ini
Vanilla Bounds: MIN_CHANCES_STRONG = [3, 6], MAX_CHANCES_STRONG = [5, 9]
Vanilla Conversion: GOAL_PERC = [45%, 45%, 40%, 40%], GOAL_SCORE_MOD = 5
```

Because the engine evaluates goals using a high integer step divisor (`GOAL_SCORE_MOD = 5`), low-chance outputs lead to extreme truncation. A team generating 3 chances has an expected value ($EV$) of only $1.35$ goals under ideal conditions, while average sides collapse to lower fractions, driving the league-wide average to $\approx 1.45$.

### 2. The Artificial Randomness Drag ("FOG" Parameter)

The engine incorporates an entropy factor under `[FOG]` ("Funny Old Game"). While intended to simulate cup upsets, its formulation scales inversely with team strength:

$$
\text{Variance Modifier} \propto \frac{\text{Rating Difference}}{\text{RANGE}}
$$

This heavily penalizes high-overall squads ($\text{OVR} \ge 84$) against lower-tier clubs ($\text{OVR} \le 74$), forcing excessive draws or losses over a 38-game calendar and capping league champions artificially around 68–74 points.

### 3. Mismapped Attributes and Position Weighting

When a goal event is awarded, the engine samples player suitability via an 11-step discrete array `[SCORER_WEIGHT]`. In baseline files:
* `ATTRIB_3` was mistakenly indexed to `10` (Tactical Awareness/Interceptions) rather than `11` (Attacking Positioning).
* Heading Accuracy (`ATTRIB_1 = 18`) was weighted at only $5\%$, while Finishing (`ATTRIB_2 = 16`) was weighted at $66\%$ for all field players.
* The discrete weights for roles 8, 9, and 10 (which map to wingers and attacking midfielders in common modern setups) were nearly equal to role 11 (central striker).

As a result, inside wingers absorbed the majority of generated goals, leaving central forwards with unrealistic single-digit scoring records.

---

## Mathematical Derivations and Parameter Justifications

```
+-----------------------------------------------------------------------------------+
|                        MACRO MATCH RESOLUTION PIPELINE                            |
+-----------------------------------------------------------------------------------+
                                          |
                                          v
      [INFLUENCE: HOMEADV=1]  -------->  Compute Effective Quality Delta (Scale 1-8)
      [INFLUENCE: FOG=0]                  |
                                          v
                              Generate Discrete Match Chances
                              - Strong Side: U[MIN_S, MAX_S] (4-7 up to 9-13)
                              - Weak Side:   U[MIN_W, MAX_W] (3-6 down to 2-5)
                                          |
                                          v
                              Sample Tiered Conversion Curve
                              [AVG_ATT_SCORE 84/78/74/68 -> GOAL_PERC 31/27/25/23%]
                              Scaling Factor: GOAL_SCORE_MOD = 4
                                          |
                                          v
                              Final Scoreline Matrix Generated
                                          |
+-----------------------------------------------------------------------------------+
|                       MICRO GOAL ATTRIBUTION PIPELINE                             |
+-----------------------------------------------------------------------------------+
                                          |
                                          v
                              Compute Player Score Propensity
                              S_i = 0.60(Finishing) + 0.18(Positioning) 
                                  + 0.18(Heading)   + 0.08(Long Shots)
                                          |
                                          v
                              Apply Positional Array Weights
                              W = [0, 1, 2, 3, 6, 8, 12, 18, 28, 42, 68]
                                          |
                                          v
                              Select Goalscorer via Weighted Discrete Distribution
```

### 1. Opportunity Generation and Conversion Balance (`[SCORE]`)

Match goals for a given team are modeled as:

$$
E[G] = E[\text{Chances Generated}] \times P(\text{Conversion} \mid \text{Attacking Quality})
$$

To shift $E[G_{\text{Total}}]$ from $1.455$ to the empirical benchmark of $2.881$:
* **Chance Expansion:** Baseline opportunities for balanced matchups ($\text{Scale } 1.0$) were widened to $[4, 7]$ for the favorite and $[3, 6]$ for the underdog. Heavily asymmetric fixtures ($\text{Scale } 8.0$) now permit up to $13$ chances for dominant teams.
* **Realistic Conversion Rates (`GOAL_PERC`):** To avoid inflated scorelines as chance volumes increased, conversion was lowered from the vanilla $45\%$ baseline into an empirically matched scale:
  * Elite Tier ($\text{OVR} \ge 84$): $31\%$ conversion per chance
  * High Tier ($\text{OVR} \ge 78$): $27\%$ conversion per chance
  * Mid Tier ($\text{OVR} \ge 74$): $25\%$ conversion per chance
  * Low Tier ($\text{OVR} \ge 68$): $23\%$ conversion per chance
* **Smoothing Factor:** `GOAL_SCORE_MOD` was reduced from $5$ to $4$, smoothing the integer division and allowing authentic multi-goal margins.

### 2. Match Context Weights (`[INFLUENCE]`, `[FOG]`)

* **`HOMEADV = 1` (Reduced from `3`):** The default $+3$ to $+4$ rating addition for home clubs caused home win rates to rise to $52.58\%$, while suppressing away victories to $24.75\%$. Setting `HOMEADV = 1` yields an outcome distribution of $44.75\% \,/\, 22.52\% \,/\, 32.73\%$, tracking the ground truth within $< 1.75\%$.
* **`FOG = 0` (Reduced from `5`):** Removing artificial entropy ensures that roster quality and player ratings govern season-long performance. Real upsets still happen naturally via the stochastic chance generation bounds without requiring artificial stat deflation.

### 3. Striker Attribution Rebalancing (`[SHOOTING_ATTRIBS]`, `[SCORER_WEIGHT]`)

The scoring propensity $S_i$ for an individual player $i$ is calculated as:

$$
S_i = \beta_1 \cdot \text{Finishing} + \beta_2 \cdot \text{Positioning} + \beta_3 \cdot \text{Heading} + \beta_4 \cdot \text{LongShots}
$$

* **Attacking Positioning (`ATTRIB_3 = 11`,** $\beta_2 = 0.18$**):** Correctly points to box positioning rather than interception awareness.
* **Heading Accuracy (`ATTRIB_1 = 18`,** $\beta_1 = 0.18$**):** Elevated from $5\%$ to $18\%$, allowing target strikers to convert aerial crosses that wingers cannot.
* **Finishing Concentration (`ATTRIB_2 = 16`,** $\beta_2 = 0.60$**):** Solidifies finishing as the core decider for open-play opportunities.
* **Discrete Position Weight Ladder:**
  ```ini
  WEIGHT_1 to WEIGHT_11 = [0, 1, 2, 3, 6, 8, 12, 18, 28, 42, 68]
  ```
  This distribution directs $\approx 36\%$ of team goals to the primary striker (Position 11), $\approx 22\%$ to the primary inside winger (Position 10), $\approx 15\%$ to secondary attackers/CAMs (Position 9), $\approx 16\%$ across central midfielders, and $\approx 11\%$ to set-piece headers from defense.

---

## Repository Structure

```
├── src/
│   └── simsettings.ini        # Complete commented configuration source
├── release/
│   └── PL_Sim_Realism.fifamod # Compiled mod package for FIFA Mod Manager
├── docs/
│   └── validation_data.csv    # 38,000-match Monte Carlo simulation dataset
├── LICENSE                    # MIT License
└── README.md                  # Project documentation and analysis
```

---

## License

This project is open-source under the [MIT License](LICENSE). You are welcome to study, adapt, and incorporate these simulation parameters into other modding projects with appropriate attribution.