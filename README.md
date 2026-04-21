# Airline-Delay-Decoded ✈️📊

> *"Delay in the U.S. aviation system is not random — it is economically costly, operationally contagious, and strategically distributed unevenly."*

---

## Project Overview

This repository contains the code, data preprocessing pipelines, and statistical analysis for our **AMS 597 Statistical Computing** graduate project at Stony Brook University.

Using **3.1 million domestic flight records** from the Bureau of Transportation Statistics across six months in 2024 (January, February, June, July, September, October), we go beyond generic delay prediction. Instead, we decode the hidden business dynamics of the U.S. aviation network through three distinct analytical pillars.

---

## Pillars

### Pillar 1 — Economics: The Make-Up Time Phenomenon

**What:** When a flight departs late, can the airline recover that lost time in the air — and what determines whether it can?

**How:** We engineered a `Make_Up_Time` variable (`DepDelay − ArrDelay`) for all delayed flights (>15 min), then applied a Random Forest with SHAP value interpretation to identify which operational factors drive recovery. A LOWESS nonlinearity check confirmed that the relationship between departure delay and recovery is threshold-based, not linear — justifying the tree-based approach over OLS.

**Findings:**
- 70.9% of delayed flights recover at least some time in the air
- Mainline carriers recover an average of **5.20 min** vs **2.83 min** for regional carriers — a gap of 2.36 min per delayed leg
- Departure delay size is the dominant driver of recovery (larger delays create stronger economic pressure to speed up)
- Random Forest: R² = 0.026, RMSE = 16.3 min — the low R² reflects inherent noise in recovery behavior, not model failure

---

### Pillar 2 — Operations: Geographic Contagion of Delays

**What:** Do delays spread through the network in a structured way, and which airports act as the primary transmission hubs?

**How:** Two-stage analysis. First, we clustered all origin-destination routes by their delay signature (K-Means, K=3) to show that delays are geographically structured, not random. Second, we used tail-number logic to track consecutive legs on the same aircraft, computing three contagion metrics per airport: Transmission Rate (% of delayed arrivals that infect the next departure), Propagation Slope (OLS coefficient of next_DepDelay ~ ArrDelay), and Propagation Factor (mean delay amplification per turnaround). Airports were then clustered into three tiers using K-Means on these metrics.

**Findings:**
- 2.59 million valid turnaround events analyzed across 295 airports
- **113 airports** classified as Outbreak Hubs (avg transmission rate 69.3%, propagation factor +2.96 min)
- **179 airports** classified as Average (56.6% transmission rate)
- **3 airports** classified as Resilient (43.6% transmission rate, propagation factor −2.06 min — they absorb delays)
- Top super-spreader: **LGB** (Long Beach) — 87.4% transmission rate, propagation factor +8.2 min per turnaround
- High-Delay route clusters show higher transmission rates than Low-Delay clusters, confirming that route structure is the seed and aircraft reuse is the carrier

---

### Pillar 3 — Strategy: Corporate Favoritism & The Regional Penalty

**What:** After accounting for the fact that some airports are simply more congested than others, do regional carriers face a longer taxi-out time than mainline carriers at the same hubs?

**How:** Three-stage modelling. (1) OLS baseline establishes the raw penalty with no airport control. (2) A mixed-effects model (`lmer`) adds a random intercept per airport `(1 | Origin)`, stripping out each airport's baseline congestion before estimating the carrier effect. (3) A favoritism interaction test adds `Carrier_Type × mainline_share` to determine whether the regional penalty is specifically larger at airports where mainline carriers hold dominant market share.

**Findings:**
- Raw regional penalty (OLS, no airport control): **+4.07 min**
- True regional penalty (mixed-effects, airport controlled): **+3.49 min** — statistically significant (t = 64.7, p < 0.001)
- **14.2%** of the raw gap is explained by airport congestion alone; **85.8%** is a structural carrier-type effect
- Airport congestion accounts for **8.8% of total taxi-out variance** (ICC = 0.088)
- Most congested airports by random intercept: JFK (+10.5 min), EWR (+9.3 min), ORD (+8.1 min), LGA (+7.6 min)
- Favoritism interaction coefficient: **+0.84 min per unit of mainline share** (p = 0.012) — the regional penalty is significantly larger at airports where mainline carriers are dominant, consistent with preferential treatment

---

## Dataset

| Field | Detail |
|---|---|
| Source | Bureau of Transportation Statistics, U.S. DOT |
| Months | January, February, June, July, September, October 2024 |
| Raw size | ~3.1 million flight records |
| After cleaning | ~3.0 million (cancelled and diverted flights removed) |
| Key columns | `DepDelay`, `ArrDelay`, `TaxiOut`, `TaxiIn`, `Distance`, `Origin`, `Dest`, `Tail_Number`, `Carrier_Type` |
| Engineered variables | `Make_Up_Time`, `Carrier_Type` (Mainline / Regional / ULCC), `Time_Block`, `Transmission_Rate`, `Propagation_Factor` |

---

## Repository Structure

```
├── data/
│   ├── flight_data_jan_2024.csv
│   ├── flight_data_feb_2024.csv
│   ├── flight_data_jun_2024.csv
│   ├── flight_data_jul_2024.csv
│   ├── flight_data_sep_2024.csv
│   ├── flight_data_oct_2024.csv
│   └── cleaned_flight_data_2024.parquet
├── notebooks/
│   ├── data_cleaning_pipeline.ipynb
│   ├── pillar1_makeup_time.ipynb
│   ├── pillar2_contagion.Rmd
│   └── pillar3_strategy.Rmd
├── charts/
│   ├── pillar1/
│   ├── pillar2/
│   └── pillar3/
├── docs/
│   ├── pillar2_contagion.docx
│   └── pillar3_strategy.docx
└── README.md
```

---

## Tech Stack & Requirements

### Python — Pillar 1 & Data Preprocessing

```
Python >= 3.9

pandas
numpy
pyarrow          # parquet read/write
scikit-learn     # RandomForestRegressor, train_test_split, metrics
shap             # SHAP value interpretation
statsmodels      # LOWESS nonlinearity check
matplotlib
seaborn
```

Install:
```bash
pip install pandas numpy pyarrow scikit-learn shap statsmodels matplotlib seaborn
```

### R — Pillars 2 & 3

```
R >= 4.2.0

arrow            # read parquet
tidyverse        # dplyr, ggplot2, tidyr, stringr, purrr
scales           # percent_format, comma
lme4             # mixed-effects modelling
lmerTest         # p-values for lmer fixed effects
broom            # tidy OLS output
broom.mixed      # tidy lmer output
factoextra       # fviz_nbclust, elbow/silhouette plots
cluster          # pam()
ggrepel          # non-overlapping text labels
ggcorrplot       # correlation heatmap
gridExtra        # grid.arrange
nycflights13     # airport lat/lon coordinates
maps             # US state outlines
igraph           # network graph structure
ggraph           # network graph visualisation
patchwork        # combining ggplot panels
```

Install:
```r
install.packages(c(
  "arrow", "tidyverse", "scales", "lme4", "lmerTest",
  "broom", "broom.mixed", "factoextra", "cluster",
  "ggrepel", "ggcorrplot", "gridExtra", "nycflights13",
  "maps", "igraph", "ggraph", "patchwork"
))
```

---

## Authors

**AMS 597 — Statistical Computing**
Stony Brook University
Instructor: Professor Silvia Sharna

| Name | SBU ID | Email |
|---|---|---|
| Aniket Pravin Kumar | 117390254 | aniketpravin.kumar@stonybrook.edu |
| Siddhi Wanzkhade | 117565113 | siddhi.wanzkhade@stonybrook.edu |
| Urvil Naik | 117624425 | urvilhemantbhai.naik@stonybrook.edu |
| Vidisha Deshpande | 117478358 | vidisha.deshpande@stonybrook.edu |
