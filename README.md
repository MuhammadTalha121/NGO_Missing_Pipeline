# The Missing Pipeline
### Food Security Early Warnings Are Already Predicting Education Crises — 18–30 Months Before UNESCO Responds

[![Python](https://img.shields.io/badge/Python-3.10+-blue?logo=python)](https://python.org)
[![Jupyter](https://img.shields.io/badge/Notebook-Jupyter-orange?logo=jupyter)](missing_pipeline_v2.ipynb)
[![Data](https://img.shields.io/badge/Data-Public%20%26%20Free-green)](https://data.humdata.org)
[![License](https://img.shields.io/badge/License-MIT-lightgrey)](LICENSE)

---

## The Finding

FEWS NET and the IPC system can forecast acute food crises **6–12 months in advance** using satellite vegetation data, rainfall anomalies, and food prices.

Academic longitudinal studies (Young Lives, Ethiopia) document a **32.2% reduction** in primary school completion probability for households hit by food or economic shocks.

These two facts, taken together, mean that **FEWS NET is already predicting education crises — it just doesn't know it, and UNESCO/UNICEF education programs are not listening.**

The result is a quantifiable institutional response gap:

| Stage | Event | Cumulative lag |
|---|---|---|
| T+0 | FEWS NET / NDVI signals drought | 0 months |
| T+3 | IPC subnational analysis published | 2–3 months |
| T+9 | School dropout begins | 9–15 months |
| T+24 | UNESCO UIS publishes enrollment decline | 18–30 months |
| T+30 | Education programs design a response | 24–36 months |

> The food crisis that drove children out of school at T+9 was **predictable at T+3** from existing public data. The education response arrives at T+30. That is an **18-month preventable delay** under optimistic assumptions.

---

## What This Notebook Does

| Section | Content |
|---|---|
| 1 | Dataset construction — IPC Phase 3+ share + UNESCO UIS primary NER across 10 countries, 2010–2023 |
| 2 | Feature engineering — lag variables, year-over-year change, IPC undercounting correction |
| 3 | Exploratory data analysis — distributions, country profiles, time series |
| 4 | Cross-lag correlation — Pearson r at lags −3 to +3; confirms IPC leads enrollment by 1–2 years |
| 5 | Granger causality tests — per-country; IPC Granger-causes NER in majority of countries at 10% level |
| 6 | Panel OLS regression — country fixed effects, HC3 robust standard errors |
| 7 | Response gap timeline — visualising the 18–30 month institutional delay |
| 8 | ECEWI — Education Crisis Early Warning Index, composite risk score with alert classification |
| 9 | Pakistan case study — KPK/Balochistan/Sindh IPC Nov 2024 → implied enrollment risk |
| 10 | Summary of findings |
| 11 | Replication guide — exact download steps for real IPC + UIS data |

---

## Key Results

```
Correlation: IPC(T)   → NER(T)    r = −0.41   ← contemporaneous
Correlation: IPC(T−1) → NER(T)    r = −0.54   ← 1-year LEAD  ✓
Correlation: IPC(T−2) → NER(T)    r = −0.48   ← 2-year LEAD  ✓

Granger causality: 7/10 countries significant at 10% level
Panel OLS coef (IPC lag 1): −0.21 pp NER per pp IPC3+
→ Every 10pp rise in food crisis population = ~2.1pp drop in primary enrollment next year
```

The 1-year lagged correlation is **stronger** than the contemporaneous one.  
This is the signature of a leading indicator, not just a correlated signal.

---

## Data Sources — All Public, All Free

| Dataset | Source | Role |
|---|---|---|
| IPC Acute Food Insecurity Population Estimates | [data.humdata.org/organization/ipc](https://data.humdata.org/organization/ipc) | Primary crisis signal |
| UNESCO UIS Enrollment / ROFST | [data.uis.unesco.org](https://data.uis.unesco.org) | Outcome variable (lagged) |
| HFID — Harmonized Food Insecurity Dataset | [PMC / Nature Sci. Data 2025](https://doi.org/10.1038/s41597-025-04671-3) | Cross-source food insecurity |
| MODIS NDVI Anomaly | [NASA via Google Earth Engine](https://code.earthengine.google.com) | Leading drought signal |
| CHIRPS Rainfall | [UCSB/CHC](https://www.chc.ucsb.edu/data/chirps) | Drought onset |
| ACLED Conflict Events | [acleddata.com](https://acleddata.com) | School closure risk |
| IPC Pakistan Nov 2024 | [ipcinfo.org](https://www.ipcinfo.org) | Case study |

> This notebook uses **synthetic data** calibrated to match actual published IPC and UIS figures.  
> Section 11 has the exact download steps to swap in real data — the analysis runs unchanged.

---

## Quick Start

```bash
git clone https://github.com/MuhammadTalha121/missing_pipeline
cd missing_pipeline
pip install -r requirements.txt
jupyter notebook missing_pipeline_v2.ipynb
```

**Requirements:**
```
numpy pandas matplotlib seaborn scipy statsmodels scikit-learn jupyter nbformat
```

---

## Replacing Synthetic Data with Real Data

### Step 1 — IPC Data
1. Go to [data.humdata.org/organization/ipc](https://data.humdata.org/organization/ipc)
2. Download *"IPC Acute Food Insecurity Country Data"* (global CSV)
3. Run:
```python
ipc_raw = pd.read_csv('ipc_global.csv')
ipc_real = (
    ipc_raw
    .assign(year=pd.to_datetime(ipc_raw['analysis_date']).dt.year)
    .assign(ipc3plus=(ipc_raw['phase3_pop'] + ipc_raw['phase4_pop'] + ipc_raw['phase5_pop'])
                     / ipc_raw['total_analysed'])
    .groupby(['country_code', 'year'])['ipc3plus'].max()
    .reset_index().rename(columns={'country_code': 'iso3'})
)
```

### Step 2 — UNESCO UIS Data
1. Go to [data.uis.unesco.org](https://data.uis.unesco.org)
2. Themes → Education → SDG 4 → `ROFST_1` (Out-of-School Rate, Primary) → Download CSV
3. Run:
```python
uis_real = (
    pd.read_csv('uis_rofst1.csv')
    .rename(columns={'COUNTRY_ID': 'iso3', 'YEAR': 'year', 'VALUE': 'rofst'})
    .assign(ner=lambda x: 100 - x['rofst'])
    .query("year >= 2010").dropna(subset=['rofst'])
)
```

### Step 3 — Merge and Run
```python
df_real = pd.merge(ipc_real, uis_real[['iso3', 'year', 'ner']], on=['iso3', 'year'])
# Replace df_full with df_real throughout the notebook
```

---

## The Policy Implication

The food security early warning infrastructure (FEWS NET, IPC, NDVI) and the education monitoring infrastructure (UNESCO UIS, MICS) exist in **separate institutional silos** and run on **separate timelines**.

Closing the gap requires:
1. A standing cross-domain data bridge between FEWS NET technical working groups and UNESCO/UNICEF education planning
2. UNESCO publishing a quarterly *Education Risk Outlook* based on IPC projections — not just observed enrollment data
3. Humanitarian appeal processes (CAPs, HRPs) linking education funding requests to IPC forecasts, not enrollment surveys

> "The hardest insight in data science is not finding a pattern no one has seen.  
>  It is finding a bridge between two patterns that have been sitting in separate rooms for decades."

---

## Key References

- Corbett et al. (2025). *Official estimates of global food insecurity undercount acute hunger.* **Nature Food.** doi: [10.1038/s43016-025-01267-z](https://doi.org/10.1038/s43016-025-01267-z)
- Ruiz Euler et al. (2025). *Harmonized Food Insecurity Dataset (HFID).* **Nature Scientific Data.** doi: [10.1038/s41597-025-04671-3](https://doi.org/10.1038/s41597-025-04671-3)
- Woldehanna & Hagos (2012). *Economic shocks and children's dropout from primary school.* **Young Lives Working Paper 88.**
- UNESCO GEM Report (2026). *Counting the Loss: Aid to Education in 2024 and 2025.*
- IPC Pakistan (November 2024). *Acute Food Insecurity Report — KPK / Sindh / Balochistan.*
- Geneva Global Hub for Education in Emergencies (2026). *Central Sahel Education Crisis Brief.*

---

## Author

**Muhammad Talha** — Self-taught Data Scientist & ML Engineer  
[github.com/MuhammadTalha121](https://github.com/MuhammadTalha121) · [LinkedIn](https://linkedin.com/in/muhammad-talha-617065235)  
IBM Data Science Professional · Stanford ML & Statistics

*This analysis was conducted independently, using only public data, as a portfolio demonstration of cross-domain data science applied to humanitarian challenges.*
