# Airline-Delay-Decoded ✈️📊

> *"Delay in the U.S. aviation system is not random — it is economically costly, operationally contagious, and strategically distributed unevenly."*

**AMS 597 — Statistical Computing | Stony Brook University | Instructor: Professor Silvia Sharna**

---

## Overview

Using **~3.5 million domestic flight records** (Jan, Feb, Jun, Jul, Sep, Oct 2024) from the Bureau of Transportation Statistics (~3.4M after cleaning), we decode the hidden business dynamics of U.S. aviation delays across three pillars.

---

## Pillar 1 — Economics: The Make-Up Time Phenomenon

**How:** Engineered `Make_Up_Time = DepDelay − ArrDelay` for delayed flights. Applied Random Forest + SHAP to identify what drives in-air recovery. LOWESS confirmed a nonlinear threshold effect before modelling.

- 70.9% of delayed flights recover some time in the air
- Mainline carriers recover **5.20 min** vs **2.83 min** for regional — a 2.36 min gap per delayed leg
- Departure delay size is the strongest driver; recovery kicks in sharply beyond ~30 min delays
- Random Forest: R² = 0.026, RMSE = 16.3 min

---

## Pillar 2 — Operations: Geographic Contagion of Delays

**How:** Clustered OD routes by delay signature (K-Means, K=3) to show delays are structured. Then used tail-number logic to track consecutive legs on the same aircraft, computing Transmission Rate, Propagation Slope, and Propagation Factor per airport. Airports clustered into three tiers.

- 2.59M turnaround events across 295 airports analyzed
- **113 Outbreak Hubs** (avg 69.3% transmission rate, +2.96 min propagation factor)
- Top super-spreader: **LGB** — 87.4% of delayed arrivals infect the next departure, amplifying +8.2 min per turnaround
- High-Delay route clusters show measurably higher transmission rates, confirming route structure seeds the contagion

---

## Pillar 3 — Strategy: Corporate Favoritism & The Regional Penalty

**How:** OLS baseline → mixed-effects model `lmer` with `(1 | Origin)` to absorb airport-level congestion → interaction test `Carrier_Type × mainline_share` to check whether the penalty concentrates at mainline-dominated hubs.

- Raw regional taxi-out penalty: **+4.07 min** (OLS, no airport control)
- True penalty after controlling for airport congestion: **+3.49 min** (p < 0.001)
- 14.2% of the raw gap is explained by airport congestion; 85.8% is a structural carrier-type effect
- Favoritism interaction: **+0.84 min** per unit mainline share (p = 0.012) — penalty grows at mainline-dominated hubs

---

## Dataset

| | |
|---|---|
| Source | Bureau of Transportation Statistics, U.S. DOT |
| Months | Jan, Feb, Jun, Jul, Sep, Oct 2024 |
| Raw records | ~3.5 million |
| After cleaning | ~3.4 million |
| Key variables | `DepDelay`, `ArrDelay`, `TaxiOut`, `Distance`, `Origin`, `Dest`, `Tail_Number` |
| Engineered | `Make_Up_Time`, `Carrier_Type`, `Time_Block`, `Transmission_Rate`, `Propagation_Factor` |

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

## Requirements

**Python**
```bash
pip install pandas numpy pyarrow scikit-learn shap statsmodels matplotlib seaborn
```

**R**
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

| Name | SBU ID | Email |
|---|---|---|
| Aniket Pravin Kumar | 117390254 | aniketpravin.kumar@stonybrook.edu |
| Siddhi Wanzkhade | 117565113 | siddhi.wanzkhade@stonybrook.edu |
| Urvil Naik | 117624425 | urvilhemantbhai.naik@stonybrook.edu |
| Vidisha Deshpande | 117478358 | vidisha.deshpande@stonybrook.edu |
