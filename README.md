# dls-microplastic-moe

Mixture-of-experts classifier and synthetic-data pipeline for the paper
_Machine learning-assisted dynamic light scattering for rapid screening of
sub-micron plastic particulates in drinking water_ (Ganpisetti, Giridharan,
Biswas and Chandankere, 2026).

This repository contains the feature matrix for the 27 measurements
reported in the paper, the 450-sample synthetic augmentation cohort, the
two trained model bundles, every Python module used to produce the
results, and the pseudocode-level reproduction of Table 4.

---

## Contents

- [Overview](#overview)
- [Key results](#key-results)
- [Repository layout](#repository-layout)
- [Installation](#installation)
- [Quick start: predict on one measurement](#quick-start-predict-on-one-measurement)
- [Full reproduction of the paper](#full-reproduction-of-the-paper)
- [Data description](#data-description)
- [Model description](#model-description)
- [Tests](#tests)
- [Troubleshooting](#troubleshooting)
- [License and citation](#license-and-citation)
- [Contact](#contact)

---

## Overview

Dynamic light scattering (DLS) is fast and cheap but cannot on its own
separate plastic particulates from natural colloids. This project shows
that a small machine learning layer built on top of routine DLS output
can screen drinking water for contamination severity at nanogram to
microgram per millilitre equivalent-polystyrene concentration.

The pipeline is:

1. Extract 13 Cumulants scalars and the 142-bin intensity-weighted size
   distribution from each DynaPro NanoStar XLSX export.
2. Engineer 62 features: distribution-shape moments, weighted
   percentiles, size-band integrated intensities, and a Stokes-Einstein
   cross-check diameter.
3. Generate 450 synthetic measurements from a multi-modal lognormal
   mixture generator and an empirical bootstrap-perturb generator.
4. Align the synthetic feature space to the real one with CORAL
   (correlation alignment).
5. Train a mixture-of-experts classifier with a random forest, XGBoost
   and LightGBM expert routed by a small MLP gating network conditioned
   on Cumulants fit-quality indicators.
6. Report class predictions with per-expert breakdown, gating weights
   and a one-class Milli-Q anomaly distance.

The full method is described in Sections 2.4 to 2.10 of the paper.

## Key results

| Evaluation | Metric | Value |
|---|---|---|
| Synthetic-to-real transfer, shape only, intensity masked (primary) | Accuracy | 0.926 |
| Synthetic-to-real transfer, shape only, intensity masked (primary) | Macro-F1 | 0.926 |
| Leave-one-group-out cross-validation | Accuracy | 0.926 &plusmn; 0.139 |
| Intensity-tertile five-fold cross-validation (consistency check) | Accuracy | 0.827 &plusmn; 0.150 |
| Wei and Chua (2026) external calibration | Spearman rho (I vs c) | 0.981 |
| Milli-Q anomaly index | PW2 z-distance | 1.94 sigma |

The classifier characterises sub-micron particulates in the 100 to
1000 nm range rather than nanoplastics below 100 nm, because the
10 to 100 nm band held less than 0.5 % of integrated intensity in all
but two samples in this cohort.

## Repository layout

```
dls-microplastic-moe/
|-- README.md
|-- LICENSE                             # MIT
|-- pyproject.toml
|-- requirements.txt
|-- .gitignore
|
|-- dlsmp/                              # main package
|   |-- __init__.py
|   |-- extract_features.py             # XLSX to scalars + 142-bin distribution
|   |-- features.py                     # engineered feature computation
|   |-- augment.py                      # synthetic data generators
|   |-- domain_adapt.py                 # CORAL alignment
|   |-- moe.py                          # mixture-of-experts classifier
|   |-- train.py                        # fit and save deployed model bundles
|   |-- evaluate.py                     # reproduce paper Table 4
|   |-- predict.py                      # command-line prediction on a new XLSX
|   |-- inverse_prediction.py           # bootstrap calibration regression
|   |-- external_validation.py          # Wei and Chua (2026) cross-check
|
|-- data/
|   |-- processed/                      # 27-measurement feature matrix
|   |   |-- dls_features.csv
|   |   |-- dls_engineered.csv
|   |   |-- size_distributions.npz
|   |   |-- anomaly_scores.csv
|   |   |-- inverse_predictions.csv
|   |   |-- inverse_calibration_params.json
|   |
|   |-- synthetic/
|   |   |-- synthetic_augmented.npz     # 450 balanced synthetic samples
|   |
|   |-- external/
|       |-- wei_chua_processed.csv      # processed reference table
|
|-- models/
|   |-- model_moe_primary.joblib        # MoE-FULL, real cohort
|   |-- model_moe_shape.joblib          # MoE-SHAPE, synthetic training
|
|-- figures/                            # all paper figures as PNG
|
|-- scripts/
|   |-- make_figures.py                 # regenerate figures 1, 2, 3
|
|-- tests/
    |-- test_features.py
    |-- test_moe.py
```

## Installation

Python 3.10 or newer is required. The package has been developed on
Python 3.13.

```bash
git clone https://github.com/ganpisetti-lab/dls-microplastic-moe.git
cd dls-microplastic-moe

python -m venv .venv
source .venv/bin/activate           # Windows: .venv\Scripts\activate

pip install -r requirements.txt
pip install -e .
```

The `-e .` line installs the package in editable mode, which makes the
`python -m dlsmp.*` module invocations below work from anywhere inside
the repository.

Alternatively, for a strict lock on library versions used in the paper:

```bash
pip install numpy==2.0 scipy==1.14 pandas==2.2 scikit-learn==1.7.2 \
            xgboost==2.1 lightgbm==4.5 torch==2.3 openpyxl==3.1 \
            joblib==1.5 matplotlib==3.9
```

## Quick start: predict on one measurement

The two deployed model bundles are already shipped in `models/`, so a
prediction on a new DynaPro NanoStar XLSX export needs one line:

```bash
python -m dlsmp.predict path/to/measurement.xlsx
```

Example output on a bottled-water sample:

```
=== BPCB2_2026-05-13-10-49-17_Untitled3_Administrator.xlsx ===
  Raw DLS scalars:
    mean intensity     =    706.8 kcps
    hydro diameter     =   1572.1 nm (Cumulants)
    Stokes-Einstein d  =   1572.6 nm (cross-check)
    polydispersity     =     78.3 %
    D50                =   1550.5 nm
    frac 10-100 nm     =     0.00 %
    frac 100-1000 nm   =     0.00 %
  PRIMARY (MoE-FULL, real cohort)  (expected accuracy 82.7%)
    -> predicted: High  | Low=0.03  Medium=0.04  High=0.93
    -> plausible (p>=0.20): [High]
       expert scalars              gate=0.943  says High
       expert bands                gate=0.056  says Medium
       expert shape                gate=0.001  says Low
  SHAPE   (MoE-SHAPE, synthetic training)  (expected accuracy 92.6%)
    -> predicted: High  | Low=0.05  Medium=0.03  High=0.92
       expert scalars_no_intensity gate=0.059  says Low
       expert bands                gate=0.055  says Low
       expert shape                gate=0.886  says High
  Milli-Q anomaly distance: 1.942 sigma
```

Batch mode on a directory:

```bash
python -m dlsmp.predict path/to/folder/
```

Machine-readable JSON output:

```bash
python -m dlsmp.predict --json path/to/measurement.xlsx
```

## Full reproduction of the paper

Every result in the paper can be regenerated from the shipped data:

```bash
# Step 1. Extract Cumulants scalars and 142-bin distributions from raw XLSX.
#         Only needed if you have new XLSX exports; the processed data are
#         already in data/processed/.
python -m dlsmp.extract_features path/to/raw/xlsx -o data/processed

# Step 2. Compute engineered features (62 total).
python -m dlsmp.features data/processed/dls_features.csv \
    data/processed/size_distributions.npz \
    -o data/processed/dls_engineered.csv

# Step 3. Generate synthetic augmentation (450 samples, deterministic seed).
python -m dlsmp.augment data/processed/dls_features.csv \
    data/processed/size_distributions.npz \
    -o data/synthetic/synthetic_augmented.npz

# Step 4. Fit and save the two deployed model bundles.
python -m dlsmp.train

# Step 5. Reproduce Table 4 of the paper.
python -m dlsmp.evaluate

# Step 6. External calibration against Wei and Chua (2026) supplementary data.
#         Requires data/external/wei_chua_si.xlsx (downloadable from
#         ACS Measurement Science Au, DOI 10.1021/acsmeasuresciau.5c00142).
python -m dlsmp.external_validation

# Step 7. Inverse prediction and per-sample equivalent-PS concentration.
python -m dlsmp.inverse_prediction

# Step 8. Regenerate the size-distribution, pipeline and CORAL figures.
python scripts/make_figures.py
```

All random seeds are fixed at 42.

## Data description

The `data/processed/` directory holds the 27 measurements reported in
the paper.

| File | Shape | Contents |
|---|---|---|
| `dls_features.csv` | 28 rows &times; 17 columns | Raw Cumulants scalars and metadata (includes C1 calibration check which is dropped downstream) |
| `dls_engineered.csv` | 27 rows &times; ~65 columns | 62 engineered features plus filename, group, replicate and Stokes-Einstein residual |
| `size_distributions.npz` | 27 &times; 142 | Intensity-, volume-, and number-weighted size distributions on the DynaPro NanoStar log-spaced grid from 0.21 nm to 19.1 um |
| `anomaly_scores.csv` | 27 rows | Per-sample IsolationForest scores against the Milli-Q baseline |
| `inverse_predictions.csv` | 27 rows | Equivalent-PS concentration in ng/mL with 95 % bootstrap CI |
| `inverse_calibration_params.json` |  | Per-polymer and combined log-log calibration coefficients from Wei and Chua (2026) |

The synthetic augmentation cohort lives in
`data/synthetic/synthetic_augmented.npz` and contains 450 samples
balanced across the three contamination classes: 240 from the empirical
bootstrap-perturb generator and 210 from the multi-modal lognormal
mixture generator.

Wei and Chua (2026) supplementary reference measurements are pre-parsed
into `data/external/wei_chua_processed.csv` (concentration, intensity,
diffusion coefficient and derived Stokes-Einstein diameter for 89 pure
polymer suspensions).

## Model description

The mixture-of-experts classifier ([`dlsmp/moe.py`](dlsmp/moe.py)) uses
three feature-subset experts:

| Expert | Algorithm | Features | Role |
|---|---|---|---|
| E1 Scalars | Random forest | Cumulants triplet + Stokes-Einstein + g1 + fit error (6) | Trusted when the Cumulants fit converges |
| E2 Bands | XGBoost | Size-band integrated intensities, intensity- and volume-weighted (6) | Robust to Cumulants failure |
| E3 Shape | LightGBM | log10(d) moments and percentiles, mode count, Shannon entropy (9) | Captures multi-modality directly |

The gating network is a two-layer MLP (5 -> 32 -> 32 -> 3) with GELU
activation and dropout (0.20, 0.10). Its five inputs are the g1
intercept, the Cumulants fit error, polydispersity, the first-peak area
and a binary flag set when polydispersity exceeds 100 %. Combination is
soft, done in log-probability space with `logsumexp`, so the model
remains differentiable end to end.

Training runs in two stages: experts are fitted on the union of the
real cohort and CORAL-adapted synthetic samples with a 1.0 / 0.5
sample-weight split, then the gating MLP is fitted with a weighted
focal cross-entropy loss (focusing parameter gamma = 2, inverse-
frequency class weights, Adam optimiser, learning rate 1e-3, weight
decay 1e-4, 600 epochs).

Full hyperparameter settings are given in Table 3 of the paper. Full
pseudocode is in ESI Appendix A.

## Tests

Nine sanity tests cover Stokes-Einstein computation, feature-vector
shapes, CORAL moment matching and MoE fit/predict behaviour:

```bash
pip install pytest
pytest tests/ -q
```

Expected output:

```
9 passed in ~10 s
```

## Troubleshooting

**`ModuleNotFoundError: No module named 'dlsmp'`**
The package was not installed. Run `pip install -e .` from the
repository root, or invoke Python with the repository root on
`PYTHONPATH`.

**`ValueError: Diameter grids differ across files`** during extraction
All XLSX exports from the DynaPro NanoStar in a given cohort must use
the same size-distribution grid. Check that all files were produced
with the same instrument configuration (log-spaced 142-bin grid,
0.21 nm to 19.1 um).

**Cumulants polydispersity above 100 %**
This is expected for multi-modal or aggregated samples and is used as
a flag rather than a rejection criterion. The model routes such
samples to the Shape expert automatically; see Section 3.3 of the
paper.

**LightGBM feature-name warning**
Harmless. It appears because we pass a plain numpy array to
`predict_proba` after a `StandardScaler` transform. All predictions
remain valid.

## License and citation

- Code is released under the MIT License; see [LICENSE](LICENSE).
- Data are released under CC-BY 4.0.

If you use this repository, please cite:

> Ganpisetti, R., Giridharan, S., Biswas, P. and Chandankere, R. (2026).
> Machine learning-assisted dynamic light scattering for rapid screening
> of sub-micron plastic particulates in drinking water.

BibTeX:

```bibtex
@article{ganpisetti2026dlsmp,
  title   = {Machine learning-assisted dynamic light scattering for rapid
             screening of sub-micron plastic particulates in drinking water},
  author  = {Ganpisetti, Ramesh and Giridharan, Sanjay and
             Biswas, Priyanka and Chandankere, Radhika},
  year    = {2026},
  note    = {Code and data: \url{https://github.com/ganpisetti-lab/dls-microplastic-moe}}
}
```

## Contact

For questions about the code or the underlying study, please contact
the corresponding author: Radhika Chandankere,
`radhika.chandankere@alliance.edu.in`.

Bug reports and feature requests may be filed as GitHub issues.
