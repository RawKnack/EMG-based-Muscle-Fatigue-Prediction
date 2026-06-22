# EMG Fatigue Pipeline

> An end-to-end surface EMG signal processing and fatigue analysis pipeline for multi-muscle resistance exercise, featuring three novel fatigue metrics — **EWFO**, **FEI**, and **DLRT**.

---

## Table of Contents

- [Overview](#overview)
- [Dataset](#dataset)
- [Pipeline Architecture](#pipeline-architecture)
- [Fatigue Detection Method](#fatigue-detection-method)
- [Novel Metrics](#novel-metrics)
- [Key Results](#key-results)
- [Output Files](#output-files)
- [Repository Structure](#repository-structure)
- [Dependencies](#dependencies)
- [Usage](#usage)
- [Visualisations](#visualisations)
- [Future Work](#future-work)

---

## Overview

This project builds a complete pipeline to detect, quantify, and compare **muscle fatigue** from surface EMG (sEMG) signals recorded during bench press exercise. The pipeline goes beyond standard onset detection — it introduces three custom metrics that capture *when*, *how efficiently*, and *how load shifts across muscles* as fatigue develops within a set.

The core question this project addresses:

> *Which muscle fatigues first, and does that ordering stay consistent across repeated sets and subjects?*

---

## Dataset

| Property | Detail |
|---|---|
| **Recording system** | Noraxon sEMG |
| **Subjects** | 5 (Sub01 – Sub05) |
| **Exercise** | Bench Press |
| **Sets per subject** | 3 (Bench_Set1, Bench_Set2, Bench_Set3) |
| **Muscles monitored** | Deltoid, Pec Major, Tricep Lateral, Tricep Medial |
| **Total records** | 60 (5 subjects × 3 sets × 4 muscles) |
| **Rep range** | 10 – 18 reps per set |

---

## Pipeline Architecture

```
Raw sEMG Signal (Noraxon)
        │
        ▼
Rep Segmentation
(detect individual repetitions from RMS envelope)
        │
        ▼
Per-Rep Feature Extraction
  ├── RMS Envelope (amplitude)
  ├── Median Frequency — MDF (spectral fatigue indicator)
  └── Dimitrov Index — FInsm5 (H/L frequency ratio)
        │
        ▼
Fatigue Onset Detection
(Mann-Kendall trend test on rolling 5-rep FInsm5 window)
        │
        ▼
Novel Metric Computation
  ├── EWFO  — Early Warning Fatigue Order
  ├── FEI   — Fatigue Efficiency Index
  └── DLRT  — Dynamic Load Redistribution Trend
        │
        ▼
Within-Subject & Across-Subject Analysis
(Kendall's W consistency, Friedman test, heatmaps)
        │
        ▼
Auto-generated Dashboards (per subject, per set)
```

---

## Fatigue Detection Method

### 1. Signal Features

**RMS Envelope** — root mean square computed per rep window, reflecting overall muscle activation amplitude.

**Median Frequency (MDF)** — the frequency that splits EMG power spectrum into two equal halves. MDF decreases monotonically during fatigue due to motor unit synchronisation and metabolic byproduct accumulation.

**Dimitrov Index (FInsm5)** — ratio of low-to-high frequency spectral moments. More sensitive than MDF alone for detecting early-stage fatigue.

### 2. Onset Detection

A **5-rep rolling window** of FInsm5 values is computed per muscle per set. The **Mann-Kendall (MK) trend test** is then applied to detect a statistically significant monotonic increasing trend (p < 0.05), which indicates confirmed fatigue onset.

The rep at which this trend first becomes significant is recorded as the **fatigue onset rep**.

```
Onset % = (onset_rep / total_reps) × 100
```

Lower onset % = muscle fatigues earlier in the set.

### 3. Within-Subject Consistency — Kendall's W

**Kendall's W** measures agreement in fatigue rank ordering across sets for the same subject. W ranges from 0 (no agreement) to 1 (perfect agreement).

| W value | Interpretation |
|---|---|
| < 0.5 | Weak consistency |
| 0.5 – 0.7 | Moderate consistency |
| > 0.7 | Strong consistency |

---

## Novel Metrics

### EWFO — Early Warning Fatigue Order

Ranks muscles by their **vulnerability to fatigue**, combining how early onset occurs with how much energy the muscle contributes.

```
FPS (Fatigue Priority Score) = onset_pct_norm / energy_pct
```

Lower FPS → muscle fatigues early relative to its workload → higher fatigue priority. Muscles are ranked 1–4 per set (1 = most at risk).

**Key finding:** EWFO ranking improved cross-set consistency from raw Kendall's W = 0.431 → **W = 0.847** (+0.417 gain).

---

### FEI — Fatigue Efficiency Index

Captures how efficiently a muscle is recruited relative to when it fatigues.

```
FEI = (1 − onset_pct) × energy_pct
```

Higher FEI → muscle contributes significant energy but fatigues late in the set → efficient recruiter. Lower FEI → fatigues early despite moderate contribution → inefficient or overloaded.

**Key finding:** Deltoid showed the highest FEI variance across subjects (IQR ~25 units), indicating highly individual recruitment strategies for this muscle.

---

### DLRT — Dynamic Load Redistribution Trend

Tracks whether each muscle's share of total EMG energy **shifts across three phases** of a set (Early / Mid / Late reps). A **Friedman test** is applied across subjects to test whether the shift is statistically significant.

```
Rep windows:
  Early  = first third of reps
  Mid    = middle third
  Late   = final third
```

Bonferroni-corrected significance threshold: α = 0.0125 (for 4 muscles).

**Key finding:** Only **Deltoid** showed a statistically significant load increase in the late phase (Friedman χ² = 7.6, p = 0.022), indicating compensatory recruitment as primary movers (Pec Major, Triceps) fatigued.

---

## Key Results

| Metric | Finding |
|---|---|
| Fatigue onset range | 18% – 83% through set depending on muscle and subject |
| Most consistent early fatiguer | Tricep Lateral (median rank ≈ 1.5 across subjects) |
| Least consistent | Deltoid (widest onset % spread, 18% – 90%) |
| EWFO consistency gain | Kendall's W: 0.431 → 0.847 |
| Significant DLRT muscle | Deltoid only (p = 0.022, Bonferroni corrected) |
| Within-subject W range | 0.30 (Sub01) – 0.61 (Sub04) — none reached strong threshold |
| MK test confirmation rate | >95% of detected onsets confirmed at p < 0.01 |

---

## Output Files

| File | Description |
|---|---|
| `emg_fatigue_master.csv` | Per-subject, per-set, per-muscle onset rep, onset %, fatigue rank, MK p-value, Kendall's W |
| `emg_fei_scores.csv` | FEI and FPS scores per muscle per set |
| `emg_ewfo_ranked.csv` | EWFO rank per muscle per set |
| `emg_dlrt_statistics.csv` | Friedman test results and early→late energy delta per muscle |
| `emg_dlrt_windows.csv` | Energy % per muscle for Early / Mid / Late rep windows |
| `emg_energy_contribution.csv` | Total energy contribution % per muscle per set |
| `Sub01_Bench_Set1_dashboard.png` | Per-subject per-set dashboard (RMS, FInsm5, MDF, onset marker) |
| `Sub01_across_set_consistency.png` | FInsm5 trajectories across sets + fatigue rank heatmap |
| `across_subjects_summary.png` | Group-level fatigue rank, onset %, Kendall's W, rank heatmap |
| `ewfo_fei_dlrt_summary.png` | Novel metrics summary — EWFO FPS, FEI distribution, DLRT bar chart |

---

## Repository Structure

```
emg-fatigue-pipeline/
│
├── EMG_Fatigue_Pipeline_v3.ipynb       # Main analysis notebook
│
├── data/
│   ├── raw/                            # Raw Noraxon .csv or .mat files
│   └── processed/
│       ├── emg_fatigue_master.csv
│       ├── emg_fei_scores.csv
│       ├── emg_ewfo_ranked.csv
│       ├── emg_dlrt_statistics.csv
│       ├── emg_dlrt_windows.csv
│       └── emg_energy_contribution.csv
│
├── figures/
│   ├── dashboards/                     # Per-subject per-set dashboards
│   ├── consistency/                    # Across-set consistency plots
│   ├── across_subjects_summary.png
│   └── ewfo_fei_dlrt_summary.png
│
├── slides/
│   └── EMG_Fatigue_Pipeline_Updated.pptx
│
└── README.md
```

---

## Dependencies

```
python >= 3.8
numpy
pandas
scipy
matplotlib
seaborn
```

Install all dependencies:

```bash
pip install numpy pandas scipy matplotlib seaborn
```

---

## Usage

Open and run the main notebook end-to-end:

```bash
jupyter notebook EMG_Fatigue_Pipeline_v3.ipynb
```

The notebook is structured in sequential sections:

1. **Data Loading & Rep Segmentation** — load raw EMG, detect rep boundaries from RMS envelope peaks
2. **Feature Extraction** — compute per-rep RMS, MDF, FInsm5
3. **Fatigue Onset Detection** — rolling MK test on FInsm5, confirm onset rep
4. **Metric Computation** — compute EWFO, FEI, DLRT, Kendall's W
5. **Visualisation** — generate all dashboards and summary figures
6. **Export** — save all processed CSVs

---

## Visualisations

### Per-Subject Dashboard
Shows RMS envelope with rep markers, FInsm5 trend with onset marker (Mann-Kendall confirmed), MDF per rep, and RMS per rep — for all 4 muscles side by side.

### Across-Set Consistency (per subject)
FInsm5 trajectories for all 3 sets overlaid per muscle, fatigue rank heatmap across sets, and onset rep trajectory across sets.

### Across-Subject Summary
Fatigue rank distribution (dot plot + median line), onset % box plots, Kendall's W bar chart per subject, and average rank heatmap (subjects × muscles).

### Novel Metrics Summary
EWFO Fatigue Priority Score per muscle, FEI distribution per muscle, and DLRT energy % bar chart (Early / Mid / Late) with Bonferroni significance markers.

---

## Future Work

- **Time series modelling (ARIMA / LSTM)** on per-rep FInsm5 to predict fatigue onset before it occurs, enabling real-time early warning
- **Expand to more exercises** (squat, deadlift, overhead press) to test EWFO/FEI generalisability
- **Increase subject pool** (n > 20) to achieve statistical power for between-subject inferential tests
- **Real-time pipeline** — port rep segmentation and MK rolling test to a streaming implementation for live biofeedback
- **Inter-session reliability** — test EWFO rank stability across sessions separated by days/weeks

---

## Citation

If you use this pipeline or metrics in your work, please cite:

```
[Your Name], "EMG-Based Muscle Fatigue Analysis with Novel Metrics (EWFO, FEI, DLRT)",
GitHub Repository, 2025. https://github.com/[your-username]/emg-fatigue-pipeline
```

---

## License

MIT License — see `LICENSE` for details.
