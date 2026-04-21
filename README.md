# Airline-Delay-Decoded ✈️📊

> *"Delay in the U.S. aviation system is not random — it is economically costly, operationally contagious, and strategically distributed unevenly."*

## 📖 Project Overview
This repository contains the code, data preprocessing pipelines, and statistical analysis for our **AMS 597 Statistical Computing** graduate project. 

Using 1.7 million domestic flight records from the Bureau of Transportation Statistics (Jan–Mar 2024), we move beyond generic "delay prediction." Instead, we decode the hidden business dynamics of the U.S. aviation network by breaking our analysis into three distinct pillars:

### Pillar 1: Economics (The "Make-Up Time" Phenomenon)
Time is money in aviation. When a flight leaves the gate late, how does it manage to speed up and arrive on time?
* **Objective:** Analyze how complex operational factors (distance, time of day, carrier) dictate an airline's ability to "make up" lost time in the air, mitigating economic loss.
* **Methodology:** Machine Learning (XGBoost/Random Forest) with SHAP value interpretation.
* **Language:** Python

### Pillar 2: Operations (Geographic Contagion of Delays)
Operations is about network flow. We map how the U.S. airspace breaks down geometrically throughout the day.
* **Objective:** Identify geographic "contagion zones" by mapping which origin-destination routes act as systemic bottlenecks that propagate morning delays into afternoon network failures.
* **Methodology:** Unsupervised Learning (K-Means / Hierarchical Clustering).
* **Language:** R

### Pillar 3: Strategy (Corporate Favoritism & The Regional Penalty)
Major airlines insulate themselves at crowded hubs, but do they push the delay penalty onto their regional partners (e.g., SkyWest, Envoy)?
* **Objective:** Accounting for the baseline strategic congestion of different hub airports, what is the true, isolated impact of carrier type (Mainline vs. Regional) on tarmac delays (Taxi-Out time)?
* **Methodology:** Mixed-Effect Modeling (`lme4`), utilizing Hub/Origin as a Random Effect.
* **Language:** R

---

## 🗄️ Dataset
* **Source:** Bureau of Transportation Statistics (U.S. Department of Transportation)
* **Timeframe:** January – March 2024
* **Size:** ~1.7 million observations
* **Target Variables:** `DepDelay`, `ArrDelay`, `TaxiOut`, engineered variables (`Make_Up_Time`, `Carrier_Type`).

---

## 🛠️ Tech Stack & Requirements
This project utilizes both Python and R. 

**Python (Pillar 1 & Data Preprocessing)**
* `pandas`, `numpy`
* `scikit-learn`, `xgboost`
* `shap`, `matplotlib`, `seaborn`

**R (Pillars 2 & 3)**
* `tidyverse` (`dplyr`, `ggplot2`)
* `lme4` (Mixed-Effects Modeling)
* `cluster`, `factoextra` (Clustering Analysis)

---

## 📂 Repository Structure
```text
├── data/               # Raw and cleaned dataset files (Not tracked if >100MB)
├── scripts/            # Python and R scripts for data cleaning
├── notebooks/          # Jupyter notebooks (Python) and RMarkdown files (R)
├── docs/               # Final PDF report and PPT presentation
└── README.md           # Project documentation 
